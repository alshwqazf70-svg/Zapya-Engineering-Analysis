# Phase 6 — Native file-transfer vs KCP call-chain findings

Date: 2026-11-09
APK: `apk/Zapya.apk`
Primary binary inspected: `libnearby_nstackx.so` (ARM64 and ARMv7)

## Executive result

The native call graph separates the file-transfer data plane from the KCP message plane.

- **File data path:** the file-specific frame builder `CreatNormalDataFrame` calls `AesGcmEncrypt`; the receive-side `WriteBlockFrame` calls `AesGcmDecrypt`. This is direct call-site evidence that AES-GCM protects file data frames in this native library.
- **File reliability path:** `HandleDFileFrame` invokes file ACK decoders, the retransmission-list handler `FileManagerSendRetranList`, and ACK/rate adaptation helpers such as `AdjustAckInterval` and `AdjustClientSendRateByAckDrop`. This is a file-specific ACK/retransmission mechanism, separate from KCP.
- **KCP path:** direct calls to `KCP_Create`, `KCP_Recv`, `KCP_UpdatePeriodic`, and `KCP_Send` are located in `DMsgSendKcpNegoResponse` and `SendDataBlock`. In `SendDataBlock`, the code selects `KCP_Send` only when a connection-mode field equals 2; the other branch calls `SocketSend`. This is evidence of a conditional KCP message/data-block path, not evidence that file payloads use KCP.
- **Native batch capability:** `NSTACKX_DFileSendFilesWithRemotePath` validates arrays of source paths and remote paths and checks the count against 501 before constructing a file-list event. The native interface can accept multiple file entries (the checked valid range is below 501); the Java call site previously traced supplies one-element arrays per invocation. These are different layers and should not be conflated.

## Evidence from ARM64 disassembly

### 1. File-frame encryption and decryption

- `CreatNormalDataFrame` at `0x3611c` calls `AesGcmEncrypt@plt` at `0x36344`.
- `WriteBlockFrame` at `0x364d4` calls `AesGcmDecrypt@plt` at `0x36600`.
- `CreateRetranDataFrame` at `0x363fc` builds a retransmission frame by reusing the normal frame builder and setting retransmission-related frame flags.

These are concrete file-frame call sites, not merely exported symbol names.

### 2. File ACK and retransmission path

- `HandleDFileFrame` at `0x333b4` calls `DecodeFileDataAckFrameUnderSecondVersion` and `DecodeFileDataAckFrame`.
- The same handler calls `FileManagerSendRetranList` at the retransmission handling branch.
- The handler also calls `FileManagerSendFileTask` and the rate/ACK helpers `AdjustAckInterval` and `AdjustClientSendRateByAckDrop`.
- File ACK generation call sites include `EncodeFileDataAckFrame` and `EncodeFileTransferDoneAckFrame`.

This supports a native file protocol with explicit acknowledgements, retransmission handling, and adaptive send pacing. It does not establish exact retry timing or window sizes; those require additional state/data-flow analysis or runtime capture.

### 3. KCP call sites

- `DMsgSendKcpNegoResponse` at `0x3f6fc` calls `KCP_Create`, `KCP_Recv`, and periodically `KCP_UpdatePeriodic`.
- `SendDataBlock` at `0x40abc` conditionally calls `KCP_Send` when the message connection mode field is 2; its alternate branch calls `SocketSend`.
- These call sites are in the DMsg/message handling area. The file frame builder and file receiver instead show AES-GCM and file ACK/retransmission helpers.

**Conclusion:** do not add KCP to UltraShare merely because Zapya ships KCP. For MalekPlay's current file-transfer protocol, prioritize reliable stream transfer, authenticated encrypted frames, bounded chunking, explicit ACKs, retry/resume, and measured pacing. A separate KCP/UDP message lane would only be justified by a demonstrated requirement.

### 4. Native multi-file API vs Java scheduler

The native `NSTACKX_DFileSendFilesWithRemotePath` validates arrays and posts a file-list event; the count check accepts values below 501. The Java scheduler call previously traced passes one source path and one remote path per native invocation and serializes its file queue. Thus:
- Native API supports a batch/list abstraction.
- The observed Java call site submits one file per invocation.
- The Java scheduler's one-active-file behavior does not prove the native library can never process multiple files internally or concurrently.

## Architecture-specific disassembly correction

The first CI disassembly pass wrote different ABI outputs to the same filename and decoded ARMv7 in the default ARM mode. That output was not suitable for instruction-level conclusions. The workflow has now been corrected to:
- name outputs using the actual ABI directory;
- decode `armeabi-v7a` as Thumb (`thumbv7-linux-android`);
- decode `arm64-v8a` as AArch64;
- include file-frame, ACK, retransmission, and DMsg/KCP call-chain symbols.

The locally checked ARM64 and Thumb disassemblies agree on the key function boundaries and call sites above. The corrected CI run is pending artifact confirmation.

## Direct implications for MalekPlay UltraShare

1. Keep the file data path independent of KCP; the current TCP-style stream + AES-GCM + SHA-256 verification design is directionally sound.
2. Do not blindly copy Zapya's native implementation or protocol. Keep UltraShare's own protocol and interoperability boundary.
3. Improve the measurable reliability features first: durable queue, verified resume identity, bounded retry/backoff, progress based on confirmed bytes, duplicate-receipt protection, and storage/error handling.
4. Add benchmark instrumentation before changing chunk size or concurrency. Compare 64/128/256/512 KiB and 1 MiB chunking on the same two devices and network; measure throughput, CPU, battery, retransmissions, and resume correctness.
5. Do not claim Zapya-like speed or superiority without repeated real-device measurements.

## Confidence

| Finding | Confidence |
|---|---|
| AES-GCM is called by file frame creation and receive code | High |
| File ACK decoding and retransmission are in the DFile handler path | High |
| KCP calls are in the DMsg/message path identified above | High for the observed direct call sites |
| The file transfer never indirectly uses KCP in any untraced callback | Not fully proven |
| Exact ACK windows, retry timers, and throughput algorithm | Not yet fully quantified |
| Java scheduler submits one-element arrays per file invocation | High |
| Native DFile API accepts a multi-entry file list | High |

## Next verification

- Confirm corrected CI artifacts contain separate ARM64 and ARMv7 Thumb disassembly files.
- Continue tracing `DFileSenderHandle`, `DFileReceiverHandle`, `FileManagerSendFileTask`, and the ACK/retransmission state transitions.
- Perform paired-device tests under packet loss, screen-off, app navigation, and process termination; static APK analysis cannot prove those runtime outcomes.
