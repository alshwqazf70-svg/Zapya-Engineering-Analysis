# Phase 4 — Application Task Model, Native Session Mapping and Remaining Unknowns

## 1. Do not confuse the app's general download database with nearby file transfer

The APK contains a general transfer/download provider (`com.dewmobile.transfer.provider.DmTransferProvider`) with a SQLite table named `transfer`. Its schema stores:
- `totalbytes`, `currentbytes`, `status`, `path`,
- `md5`, `net`, `priority`,
- directory sequence fields `fileseq_int` and `fileseq_currentbytes`,
- device/user metadata and other application fields.

The model `com.dewmobile.transfer.api.l` reads and updates current bytes, total bytes, destination path, status, and directory sequence. It also deletes or renames partial paths and sidecar `.dm` files in cleanup paths.

However, other code in this package (including `DmDownloadThread`) is clearly an HTTP/download implementation and contains redirect retry and HTTP resume logic. Therefore the database schema and `.dm` sidecar alone must not be cited as proof that Nearby/Wi-Fi file sends resume after interruption. The nearby transfer manager's own native API path must be traced separately.

## 2. Nearby file-send task granularity

In the recovered Nearby manager:
- The public send entry branches on data type (bytes/file/stream).
- File mode stores a current file task and negotiates one file data server/client session.
- The actual native submission in the inspected call site passes a single source path and a single remote identifier in one-element arrays.
- The manager tracks the resulting native transfer ID and maps it to a task ID.
- Cancellation checks the active file transfer ID and calls native `dFileCancelTransfer`.
- Session close removes the session mapping and calls `dFileClose`.

Conclusion: this call path proves per-file native submissions and explicit cancellation/session cleanup. It does not establish parallel native file transfers or a full multi-file queue policy. The app may schedule files at a higher layer; that scheduler needs separate call-graph inspection.

## 3. Session registry and callback routing

The manager registers file sessions and byte sessions separately with the native session registry:
- type 1 = file,
- type 2 = byte/message.
It stores session IDs separately and associates them with the transfer manager instance.
Native callback records contain session ID, client session ID, transfer ID, message type, error code, total bytes, bytes transferred, file count/name list, and optional byte user data.

This separation is a useful architecture lesson: ephemeral transport sessions and durable user-visible transfer jobs should be separate identities. UltraShare should persist a stable job ID and map it to one or more ephemeral connection/session IDs.

## 4. Specific conclusions we can safely make now

- Zapya's Nearby transfer path has a request/response control stage, then opens a separate data endpoint.
- The file and byte modes use native NSTACKX sessions; stream mode has a distinct Java socket pipeline.
- WLAN and P2P endpoint candidates are represented separately and selected according to the P2P-connected state.
- Native engine symbols show KCP, ACK, heartbeat, retry bookkeeping, adaptive rate-control and AES-GCM/DTLS/TLS components.
- Java exposes native cancel/close APIs and maps callbacks to transfer IDs.
- The app has a separate persistent transfer/download database, but this alone does not prove Nearby resume behavior.

## 5. Remaining research, prioritized

### Priority A — native engine call graph
Use a disassembler/decompiler (Ghidra/IDA or equivalent) on `libnearby_nstackx.so` to identify:
- which exported DFile/DMsg functions invoke KCP,
- where AES-GCM encryption/decryption is applied,
- how the share-key input is transformed and whether a fresh session key is negotiated,
- how ACK and retry queues are updated,
- whether file chunks have sequence/offset metadata and whether receiver state is persisted.

### Priority B — Java app-level scheduler
Trace callers of `WifiDataTransferManager.c(C0363md)`, file task creation, callback-to-provider updates, and next-file scheduling. Establish whether multiple files are sequential, bounded parallel, or one session with multiple file entries.

### Priority C — runtime validation on two authorized devices
Capture a controlled send/receive session:
- discovery start/stop and endpoint callback timing,
- control request/response and selected interface,
- data socket/UDP/TCP flows,
- behavior when WLAN/P2P disconnects,
- behavior when app screen closes or process is killed,
- retry after packet loss and whether a partially received file can resume,
- hash equality at both ends.

## 6. UltraShare architecture decision

Do not replicate the proprietary NSTACKX protocol. Implement an original engine with:
- a versioned authenticated control protocol,
- peer identity and explicit pairing approval,
- WLAN + Wi-Fi Direct candidate abstraction and route verification,
- a foreground transfer service with durable queue,
- a bounded file scheduler (start sequentially, then benchmark safe parallelism),
- streaming I/O and bounded buffers,
- authenticated encryption with fresh session keys,
- per-chunk integrity and final SHA-256 verification,
- atomic partial-file commit and a formally specified resume checkpoint protocol,
- deterministic retry/backoff, cancellation and cleanup,
- process-death recovery tests.

**Important limitation:** this phase is still static reverse engineering of an older supplied APK. No claims are made about current Zapya versions, runtime performance, or exact native wire behavior.
