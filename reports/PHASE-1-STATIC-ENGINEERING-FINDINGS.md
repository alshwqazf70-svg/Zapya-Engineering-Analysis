# Zapya APK — Static Engineering Findings (Phase 1)

**Analysis run:** 42603879581  
**Analyzed APK:** `apk/Zapya.apk` in this private repository  
**Package:** `com.dewmobile.kuaiya.play`  
**Analysis mode:** Static inspection of decoded Android manifest, JADX-recovered Java, APK native libraries, exported ELF symbols, and printable native strings.

> This report records evidence from the analyzed APK. It does not claim runtime verification, full native decompilation, or proof that every discovered subsystem is used by every transfer mode. The APK is an older build (ZIP entry timestamps observed as 2021-04-09 in the initial inventory); behavior may differ in current Zapya releases.

## 1. Pipeline result and limitations

- GitHub Actions run completed successfully: all 10 analysis steps succeeded.
- The original APK SHA-256 was verified against the repository inventory before decoding.
- APKTool decoded the manifest and resources.
- JADX recovered **10,832 Java source files** from the DEX files.
- JADX still reported **33 decompilation errors** and returned exit code 3. The workflow was deliberately changed to continue only when actual Java source files were recovered. Therefore this is a substantial but not perfect decompilation.
- Native libraries were extracted and inspected with `strings` and `readelf -Ws`; native machine code was not fully decompiled or dynamically traced.
- Artifact: `zapya-analysis-results`, GitHub Actions artifact ID `13666489258`. It expires after the configured seven-day retention period, so preserve the local/private report and rerun the workflow if the artifact expires.

## 2. High-confidence architecture findings

### 2.1 Wi-Fi discovery is delegated to Huawei NSTACKX native code

Evidence:
- Java wrapper: `com.huawei.nstackx.core.NstackxCoreDiscover`.
- Its static initializer loads `nearby_nstackx`.
- Native methods include `discoveryInit`, `discoveryRegisterDevice`, `startScan`, `startBroadcasting`, `stopScan`, `stopBroadcasting`, and `discoveryDeInit`.
- `com.huawei.hms.nearby.C0351kb` performs the scan setup sequence: initialize discovery, register the local device, then start scanning; the callback implementation `C0336hb` forwards found-endpoint bytes to a message handler.
- `com.huawei.hms.nearby.C0341ib` performs the advertiser sequence: initialize, register device, start broadcasting; stop performs stopBroadcasting and deinitialization.

Interpretation: discovery/advertising is not merely a Java UDP broadcast loop. A JNI bridge enters a dedicated native discovery engine. The precise over-the-air packet format and discovery transport require further native analysis or runtime capture.

### 2.2 File transfer and byte/message transfer are separate native sessions

Evidence:
- Java wrapper: `com.huawei.nstackx.core.NstackxCoreTransfer`.
- It exposes distinct native APIs for file sessions (`dFileClient`, `dFileServer`, `dFileSendFiles`, `dFileSendFilesWithRemotePath`, `dFileSetStoragePath`, `dFileCancelTransfer`, `dFileClose`) and message/byte sessions (`dMsgClient`, `dMsgServer`, `dMsgSendData`, `dMsgCancelTransfer`, `dMsgClose`).
- `com.huawei.nstackx.service.transfer.wifi.c` creates and tracks these independently. It stores file and byte session IDs separately, registers them in separate maps, and tracks transfer IDs.
- For byte payloads, the Java code prepares a payload and calls `dMsgSendData`; for files it calls `dFileSendFilesWithRemotePath` with a source path and remote path/identifier.
- The server side creates a byte-message server or a file server and returns the selected listening port to the peer through a control message.

Interpretation: the design separates control/metadata messages from bulk file transfer. This is a key design pattern worth adopting in an independent UltraShare implementation.

### 2.3 Connection negotiation can choose WLAN or P2P

Evidence in `com.huawei.nstackx.service.transfer.wifi.c`:
- The peer connection candidates are represented as entries with type 1 (WLAN) and type 2 (P2P).
- The selection method prefers P2P when a local preference/flag is enabled and a P2P candidate exists; otherwise it uses WLAN if available.
- If neither usable candidate exists, it fails the connection selection.
- The chosen candidate supplies the address used when creating the native transfer server/client.

Interpretation: the inspected transfer manager supports at least two connection candidate types and makes a policy choice between P2P and WLAN. This is stronger evidence than merely finding Wi-Fi Direct-related keywords. The exact way the P2P network is formed and the conditions controlling the preference still need call-chain analysis.

### 2.4 A separate LocalOnlyHotspot service exists

Evidence in `com.dewmobile.sdk.api.DmLocalHotspotService`:
- It calls `WifiManager.startLocalOnlyHotspot`.
- On success, it retains a `LocalOnlyHotspotReservation` and broadcasts the generated SSID and pre-shared key using the internal action `local_host_action`.
- It closes the reservation when stopping.
- On Android 8+ it starts as a foreground service and calls `startForeground` with a supplied notification.
- The service is declared in the manifest as a non-exported service in a separate process (`:lhs`).

Interpretation: the APK includes an explicit local-hotspot lifecycle implementation. This service should not automatically be assumed to be the only transport path; the NSTACKX transfer manager separately handles WLAN/P2P candidates.

### 2.5 Native library contains a substantial transfer/discovery engine

The exported symbol table of `libnearby_nstackx.so` contains symbols including:
- Discovery: `NSTACKX_Init`, `NSTACKX_RegisterDevice`, `NSTACKX_StopScan`, `NSTACKX_StopBroadcasting`.
- File transfer: `NSTACKX_DFileClient`, `NSTACKX_DFileServer`, `NSTACKX_DFileSendFiles`, `NSTACKX_DFileSendFilesWithRemotePath`, `NSTACKX_DFileSetStoragePath`, `NSTACKX_DFileCancelTransfer`, `NSTACKX_DFileClose`.
- Message transfer: `NSTACKX_DMsgClient`, `NSTACKX_DMsgServer`, `NSTACKX_DMsgSendData`, `NSTACKX_DMsgCancelTransfer`, `NSTACKX_DMsgClose`.
- Progress/state: `FileGetBytesTransferred`, `FileListGetBytesTransferred`, transfer-done frame/ACK helpers.
- Network/runtime: TCP socket helpers, UDP socket helpers, heartbeat client/server functions, WLAN/P2P send-rate adjustment functions, and Wi-Fi station information helpers.

The binary also contains symbols/data associated with KCP (for example `IKCP_CMD_ACK` and `IKCP_ACK_FAST`). This is an investigation lead suggesting a reliable-UDP component may be included; symbol presence alone does **not** prove that a particular Zapya transfer uses KCP.

## 3. Transfer lifecycle reconstructed from Java

The evidence supports this provisional flow:

1. **Discovery:** initialize NSTACKX, register local device metadata, start scan or broadcasting, and forward endpoint-found callback bytes to the Java-side handler.
2. **Peer negotiation/control:** exchange control messages using the Nearby/NSTACKX service layer. A request identifies the desired transfer type (file, bytes, or stream).
3. **Transport selection:** obtain candidate endpoints; choose P2P when preferred and available, otherwise WLAN.
4. **Server preparation:** create the appropriate native file/message server or a Java stream `ServerSocket`; obtain its dynamically assigned port.
5. **Port exchange:** send the chosen server port and candidate information back to the peer in a control response.
6. **Client preparation:** create a matching native file or byte client session on the peer.
7. **Payload transfer:** use the file API for files and the message API for byte payloads. Session IDs and transfer IDs are tracked separately.
8. **Progress and completion:** native callbacks are converted into Java callback messages containing session ID, transfer ID, total bytes, bytes transferred, filenames, message type, and error code.
9. **Cancellation/cleanup:** native cancel and close APIs are exposed and used by the Java transfer manager.

The exact handshake message schema, cryptographic authentication, encryption mode, congestion-control algorithm, chunk framing, and retry/resume semantics have not yet been fully reconstructed.

## 4. Performance and reliability clues

Observed evidence:
- The Java stream server binds a dynamically assigned port (`new InetSocketAddress(0)`).
- It configures socket receive/send buffers to 8 MiB in the inspected stream server.
- Native symbols include heartbeat client/server, transfer completion frames and ACKs, byte counters, and separate WLAN/P2P send-rate adjustment routines.
- The Java transfer manager handles distinct session IDs, transfer IDs, and transfer progress callbacks.
- A transfer request/response flow chooses a connection candidate and communicates server port details to the peer.

These are useful architecture clues, not measured performance claims. No throughput benchmarks or packet-level traces were performed.

## 5. Manifest observations

The decoded manifest declares permissions related to Wi-Fi state/configuration, network state, multicast changes, location, Bluetooth, foreground service, wake lock, storage, and internet access. It also declares a non-exported local hotspot service and AndroidX WorkManager infrastructure.

A permission declaration does not prove that a permission is used by the file-transfer path. Android version and runtime permission behavior must be considered separately when designing UltraShare.

## 6. What this means for MalekPlay / UltraShare

Recommended independent design (do not copy Zapya code, proprietary assets, or protocol implementation):

- **Discovery module:** own device identity, capability advertisement, scan lifecycle, duplicate suppression, and timeout handling.
- **Negotiation/control channel:** authenticated peer handshake, capability exchange, connection candidates, selected transport, dynamic port exchange, and explicit transfer acceptance.
- **Transport abstraction:** WLAN and Wi-Fi Direct/P2P implementations behind one interface, with a deterministic fallback policy.
- **Separate data planes:** small control/message channel separated from bulk file streaming.
- **Transfer engine:** bounded buffers, streaming I/O, queue of multiple files, per-file progress, cancellation, and resource cleanup.
- **Integrity:** per-file cryptographic hash (e.g. SHA-256) and final verification before marking a file complete.
- **Reliability:** explicit state machine, bounded retries, timeout policy, atomic temporary-file writes, and resumability only after a verified chunk/offset protocol is designed.
- **Lifecycle:** foreground service for active transfers where required by Android, notification actions, process-death recovery, and persisted transfer metadata.
- **Security:** authenticated pairing, session keys, encryption in transit, path traversal protection, safe filenames, and strict receiver-side storage policy.

## 7. Remaining research tasks

1. Trace the Java call graph from the app's send/receive UI into the Nearby discovery and transfer manager.
2. Inspect the control-message classes and identify message types, capability negotiation, and candidate endpoint fields.
3. Trace the actual P2P group/hotspot establishment paths and their relationship to NSTACKX.
4. Inspect native strings and disassembly around transfer setup, framing, ACK, heartbeat, and congestion functions.
5. On two devices owned/authorized by the user, capture a controlled transfer and compare observed traffic to the static hypotheses.
6. Document whether resume/retry is implemented in the native engine, Java wrapper, or a higher-level application layer.

## 8. Confidence ledger

| Finding | Confidence | Basis |
|---|---|---|
| NSTACKX native discovery wrapper is used by the Nearby SDK | High | Java JNI wrapper and direct call sites |
| Separate native file and byte/message session APIs exist | High | Java declarations and concrete call sites |
| Transfer manager chooses between WLAN and P2P candidates | High | Decompiled candidate selection method |
| LocalOnlyHotspot service exists and manages SSID/password reservation | High | Decompiled Android service |
| Native engine includes heartbeat and ACK-related machinery | High | Exported symbols |
| KCP is used for every or any specific transfer path | Unconfirmed | KCP-related symbols only |
| Exact on-wire protocol, encryption, and resume algorithm | Unknown | Not established by this static phase |

---
**Important:** This is a technical analysis of an APK supplied for the user's own research. The independent UltraShare implementation should be written from scratch and should not include Zapya's APK, Java source, native libraries, copied protocol implementation, or proprietary assets.
