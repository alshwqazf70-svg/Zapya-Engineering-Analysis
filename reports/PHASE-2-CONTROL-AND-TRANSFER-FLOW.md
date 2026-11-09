# Zapya APK — Control Negotiation and Receive-Path Notes (Phase 2)

This document extends the Phase 1 static report using recovered Java source from the same private APK analysis artifact. It describes observed design patterns only; it is not a recommendation to reproduce Zapya's proprietary wire format.

## 1. Control negotiation is a distinct control plane

The recovered class `com.huawei.hms.nearby.C0417xc` is a TLV-style control message named in logs as `CreateSocketTlv`. The transfer manager uses it to request a data endpoint and to return endpoint candidates.

Observed command semantics in the Java parser/call sites:
- Command 1: request a server endpoint for a requested transfer type.
- Command 2: response containing one or more connection candidates and the selected server port.
- Transfer type 1 is byte/message data.
- Transfer type 2 is file transfer.
- Transfer type 3 is stream transfer.
- Candidate records carry a candidate type, address string, and port.
- The byte/message request also exchanges a native engine capability bitmap.

The control request is sent before the bulk data channel is created. The receiver prepares the matching server, then sends its endpoint candidate list and port back. The requester selects a candidate and creates the matching client session. This confirms a two-stage architecture: control negotiation first, then the data socket/session.

For UltraShare, implement a clean, documented, versioned control protocol of our own. Do not copy Zapya's TLV packet layout, command IDs, or proprietary encoding.

## 2. Endpoint candidate selection

In `com.huawei.nstackx.service.transfer.wifi.c`:
- Candidate type 1 is logged and handled as WLAN.
- Candidate type 2 is logged and handled as P2P.
- The selection method chooses P2P when the local P2P preference flag is enabled and a P2P candidate is present; otherwise it falls back to WLAN.
- The same candidate-selection method is used by the server-response and peer-response paths.

This suggests transport choice is negotiated from available endpoints rather than assuming one fixed IP/port. A robust independent implementation should represent each candidate with transport kind, address, port, reachability result, and priority, then apply a documented fallback policy.

## 3. Separate payload modes

The transfer manager branches by transfer type:
- File: creates a native file server/client, sets the receive storage path, and submits file paths plus a remote identifier.
- Bytes: creates a native message server/client and sends byte payloads through the message API.
- Stream: creates a Java `ServerSocket` path, wraps the accepted socket in the Nearby stream abstraction, and attaches the stream handler.

The stream server binds to an ephemeral port and configures large socket buffers (8 MiB receive/send buffer requests in the inspected Java path). This is a configuration request; Android/Linux may clamp the actual buffer sizes.

## 4. File receive path and task metadata

The native file server path in `com.huawei.nstackx.service.transfer.wifi.c`:
- Ensures the directory `/sdcard/Download/Nearby/` exists.
- Creates a native file server.
- Sets the server's storage path.
- Reads the assigned port and registers the server session before returning the endpoint to the peer.

The higher-level transfer item model `com.dewmobile.transfer.api.l` stores total bytes, current bytes, destination path, status, file sequence, network marker, and other task metadata. Its update method reads fields such as `currentbytes`, `totalbytes`, `status`, `path`, and `fileseq_int` from content values. It also manages a companion `.dm` file and renames an existing partial path during cleanup paths.

This indicates the application has a persistent transfer-task layer above the native transport. However, the presence of a current-byte field and a sidecar file does not by itself prove that interrupted transfers can resume safely from an arbitrary byte offset. Resume semantics still require tracing the native API and task recovery logic.

## 5. Callback and progress model

The native callback model `NstackxCoreMsg` carries:
- session ID and transfer ID,
- message type and error code,
- total bytes and bytes transferred,
- file names/count,
- optional byte user data.

The wrapper dispatches callbacks for received bytes, byte transfer progress, file transfer progress, and connection events. The higher-level manager maps these callbacks to transfer-task identifiers and reports progress to the application.

For UltraShare, retain this separation: transport session IDs should not be the same as persistent user-facing transfer IDs. Persist the latter and map transient sessions to them.

## 6. Multi-file scheduling — current evidence and uncertainty

The recovered native API accepts arrays of source paths and remote paths. However, the inspected Java call site submits a one-element array for the current file, and the manager stores one current file item in that path. The application also has a higher-level transfer-task model and callbacks for task activation/update.

Therefore, multi-file transfer is supported at the application/task level, but the exact concurrency policy (strictly sequential vs bounded parallel sessions) is not established from this evidence alone. Do not infer parallel transfer from the array-based native method signature.

## 7. UltraShare engineering implications

Recommended original implementation:
1. Define a versioned handshake message with protocol version, device ID, capabilities, nonce, and authentication proof.
2. Exchange a list of transport candidates and select based on measured reachability and policy.
3. Establish an authenticated control channel before exposing a data listener.
4. Negotiate file vs stream vs small-message transfer explicitly.
5. Keep persistent transfer IDs separate from ephemeral socket/session IDs.
6. Use a bounded scheduler with a configurable concurrency limit and backpressure.
7. Write incoming files to a temporary path; verify expected size and SHA-256; atomically rename only after verification.
8. Persist progress only at known-safe checkpoints. Resume only if both peers agree on a verified chunk boundary and the receiver can validate the partial file.
9. Keep cancellation, retry, timeout, cleanup, and process-recovery states explicit and testable.

## 8. Confidence notes

| Observation | Confidence |
|---|---|
| Control request precedes data-server endpoint exchange | High |
| Commands distinguish request and response | High |
| Transfer modes distinguish file, bytes, and stream | High |
| WLAN/P2P candidates include address and port | High |
| Java path selects P2P preferentially when enabled, otherwise WLAN | High |
| Higher-level persistent transfer metadata includes current/total bytes and status | High |
| Sidecar file proves resumable transfer | Not established |
| Native array API proves simultaneous multi-file concurrency | Not established |

**Static-analysis boundary:** no live traffic capture, device-to-device experiment, or full native-code decompilation has yet been performed.
