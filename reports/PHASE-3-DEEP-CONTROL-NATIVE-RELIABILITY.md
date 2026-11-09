# Phase 3 — Deep Control-Plane, Native-Engine and Reliability Findings

**Source:** successful private analysis artifact from run `42603879581`; recovered Java sources and ELF strings/symbols.  
**Scope:** static analysis only. No live device traffic or runtime instrumentation.

## 1. Control-plane handshake (Java evidence)

The transfer manager is `com.huawei.nstackx.service.transfer.wifi.c`. Its listener handles `C0417xc` control messages with command 1 (request) and command 2 (response).

Observed request flow:
1. Sender calls `A(type)`, creating command 1 with the requested data type.
2. Receiver handles command 1 in `o(C0417xc)`.
3. Receiver verifies a share key exists, gathers endpoint candidates, selects a usable address, and prepares a matching data server.
4. Receiver obtains the server's dynamically assigned port and returns command 2 with candidate address/port entries.
5. Requesting peer handles command 2 in `y(C0417xc)`, selects a candidate, and creates the corresponding client.
6. Native/stream connection callbacks signal readiness; only then is payload submission performed.

The TLV class `com.huawei.hms.nearby.C0417xc` confirms:
- command 1 = request; command 2 = response.
- type 1 = bytes, type 2 = file, type 3 = stream.
- Each response candidate record is 19 bytes: one type byte, a 16-byte address field, and a 2-byte little-endian port.
- A zero tail byte terminates the candidate list.
- The byte-data request/response also carries an engine capability bitmap.

These are reverse-engineered details of the supplied APK. UltraShare must use its own independently designed and documented protocol, not this packet layout or command numbering.

## 2. Candidate discovery and selection

The transfer manager constructs candidate entries from:
- P2P address: obtained from a P2P connection helper when `mP2pConnected` is true.
- WLAN address: enumerates network interfaces whose names begin with `wlan` and selects an IPv4 address.

Selection:
- If a P2P candidate exists and the manager's P2P-connected flag is true, it chooses P2P.
- Otherwise it chooses WLAN if present.
- If neither is available, it fails selection.

This code does not show a general active reachability probe across every candidate; it shows a policy based on connection state and candidate availability. A modern implementation should verify candidate reachability and account for Android routing/interface selection.

## 3. Data-mode setup details

### File mode
- Receiver ensures `/sdcard/Download/Nearby/` exists.
- Starts `dFileServer(address, 0, keyBytes, keyLength)`, where port 0 requests an OS-assigned port.
- Configures native storage path with `dFileSetStoragePath`.
- Reads the server port using `dFileGetServerPort`.
- Sender creates `dFileClient(address, port, keyBytes, keyLength)`.
- A file task submits a one-element path array and a one-element remote-path/identifier array through `dFileSendFilesWithRemotePath`.

### Byte mode
- Receiver creates `dMsgServer(address, 0, shareKeyBytes, keyLength, ...)`.
- It selects a common native byte engine by intersecting client and server capability bitmaps.
- Sender creates `dMsgClient(address, port, shareKeyBytes, keyLength, ...)`.
- Payload is wrapped by a Java helper before calling `dMsgSendData`.
- A bounded Java queue of up to 1,000 data IDs is used to suppress repeated IDs.

### Stream mode
- Uses a separate Java stream pipeline and socket server rather than the native file/byte API.
- Client/server pipeline callbacks update connection state.
- Stream frames carry a data ID and message type; callbacks acknowledge/complete the corresponding stream operation.

## 4. Native reliability and transport evidence

The exported symbols and strings in `libnearby_nstackx.so` contain:
- KCP APIs: `KCP_Create`, `KCP_Config`, `KCP_Send`, `KCP_Recv`, `KCP_UpdatePeriodic`, `KCP_Destroy`, and embedded `ikcp_*` symbols.
- File acknowledgement functions: `EncodeFileDataAckFrame`, `DecodeFileDataAckFrame`, `HandleFileDataAckFrame`, `EncodeFileTransferDoneAckFrame`.
- Byte-message acknowledgement: `CreateDMsgAckUnit`, `NSTACKX_DMsgSendDataWithPeerACK`.
- Heartbeat: `StartHeartBeatClient`, `StartHeartBeatServer`, `StopHeartBeatClient`, `StopHeartBeatServer`, `HeartbeatClientMainLoop`.
- Retry/rate-control leads: `UpdateAllTransRetryCount`, `AddRecordToUnAckInfo`, `GetClientUnAckMinTimeout`, `AdjustAckInterval`, `AdjustClientSendRateByAckDrop`, `AdjustWlanSendRate`, `AdjustP2PSendRate`.
- Crypto-related symbols/strings: `AesGcmEncrypt`, `AesGcmDecrypt`, `CreateEncryptDMsgDataUnit`, OpenSSL AES-GCM primitives, and CoAP DTLS/TLS support strings.

Interpretation:
- KCP is not merely a generic word in the binary; its API and embedded ikcp implementation are present in the native engine. This is strong evidence that a KCP transport component is included, but static symbol presence alone does not establish which specific transfer types or sessions use it.
- ACK, retry, heartbeat, and adaptive send-rate routines are also present. This strongly suggests native reliability and congestion/flow management components.
- AES-GCM and DTLS/TLS primitives are present. This does not prove the exact cipher suite, key derivation, authentication model, or that all file payloads are encrypted with AES-GCM. The Java share key is truncated to at most 16 UTF-8 characters before being passed into the native APIs; its use and security properties need native call/data-flow tracing.

## 5. Important unresolved security questions

1. Is the share key a pre-shared secret, a password-derived key, a session seed, or merely an input to native negotiation?
2. Does the native engine derive fresh session keys/nonces? How are nonces unique?
3. Is peer identity authenticated, or does possession of the share key alone authorize the transfer?
4. Are file and byte channels encrypted identically?
5. Are control TLVs authenticated/integrity-protected?
6. Are replay, downgrade, and malicious endpoint candidates rejected?
7. Does KCP carry encrypted payloads, or is it used only for a subset of data/control traffic?

No conclusion should be made until the native data flow is traced or runtime behavior is measured.

## 6. Retry/resume distinction

The binary contains retry counters, unacknowledged-data tracking, ACK frames, transfer-done ACKs, and progress counters. These support packet/frame-level reliability and retransmission.

They do **not** by themselves prove user-visible resume after app/process termination. Resume requires durable mapping of a transfer to a partial destination, a validated offset/chunk boundary, and a peer protocol to continue from that boundary. The current static evidence is insufficient to claim safe cross-process resume.

## 7. Lifecycle and Android background behavior

- Manifest declares a non-exported `DmLocalHotspotService` in a separate process (`:lhs`).
- On Android O+ its start path expects a notification and invokes `startForeground`.
- It uses `WifiManager.startLocalOnlyHotspot`, stores the reservation, broadcasts SSID/password internally, and closes the reservation when stopping.
- The service handles location permission/location-mode failures explicitly.
- Manifest includes WAKE_LOCK and FOREGROUND_SERVICE permissions and WorkManager infrastructure, but those declarations do not prove that the core file-transfer session itself is persisted or resumed by WorkManager.

## 8. UltraShare design requirements derived from this phase

- Use a control channel with authenticated, versioned messages, separate from bulk data.
- Advertise multiple candidate interfaces, but verify routes/reachability before selecting one.
- Negotiate transport capabilities explicitly and have a deterministic fallback.
- Use an independent secure session handshake; never treat a truncated user share code as a cryptographic session key.
- Apply authenticated encryption (for example modern AEAD) with fresh per-session keys/nonces and replay protection.
- Use explicit framing, sequence numbers, ACK policy, bounded retransmission, and congestion control.
- Persist transfer IDs and checkpoints; distinguish packet retry from resumable file transfer.
- Use temporary files, validate paths, verify final SHA-256, and atomically commit completed files.
- Keep foreground-service lifecycle and transfer state persistence separate; process death recovery must be tested.
- Bound concurrency and buffers to prevent memory exhaustion and provide backpressure.

## 9. Evidence confidence table

| Finding | Confidence | Qualification |
|---|---|---|
| Request/response control handshake exists | High | Direct Java call path |
| Candidate list includes WLAN/P2P type, address, port | High | Java model and TLV parser |
| P2P chosen when connected/preferred; WLAN fallback | High | Selection method |
| File/bytes/stream use distinct setup paths | High | Transfer manager branches |
| KCP implementation included in native engine | High | Native exported symbols and strings |
| ACK/retry/heartbeat/adaptive-rate machinery included | High | Native symbol/string names |
| AES-GCM crypto routines included | High | Native strings/symbols |
| All payloads encrypted with AES-GCM | Unknown | No complete crypto data-flow trace |
| User-visible interrupted-file resume | Unknown | Not proven by counters/ACKs |
| Specific transport uses KCP | Unconfirmed | Need native call graph/runtime trace |

**No Zapya code, binary, protocol implementation, or proprietary assets are to be copied into MalekPlay.**
