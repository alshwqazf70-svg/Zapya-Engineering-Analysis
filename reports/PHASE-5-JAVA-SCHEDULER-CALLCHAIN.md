# Phase 5 — Java Scheduler Call-Chain Findings

**Source APK:** `apk/Zapya.apk` (the exact APK whose SHA-256 is recorded in `inventory/SHA256.txt`).  
**Evidence source:** JADX output from workflow run `42603879581`, 10,832 recovered Java files.  
**Scope:** static Java call-chain review. This phase does not claim native instruction-level proof or runtime behavior.

## 1. Scheduler owner and independent queues

Class: `com.huawei.hms.nearby.C0358ld` (log tag: `DataTransferEngine`).

The class owns three independent `LinkedBlockingQueue<C0363md>` instances:
- `a`: data type 1 (file)
- `b`: data type 3 (stream)
- `c`: data type 2 (bytes/message)

The `a(C0363md, Xc)` entry point dispatches each task by its data type. Each queue has a separate synchronization object and in-flight boolean:
- file queue: `e` / `k`
- stream queue: `f` / `j`
- bytes queue: `g` / `l`

The queue admission check rejects a queue whose size reaches 100. This is an observed guard, not proof that every application-level multi-file selection is limited to 100 items across all layers.

## 2. File queue scheduling

The file queue worker is `c()`:
1. Synchronize on `e`.
2. Return if `k` is already true.
3. Poll exactly one `C0363md` from file queue `a`.
4. Set `k = true`.
5. Submit `RunnableC0343id` for that single task.

`RunnableC0343id.run()` invokes `C0358ld.b(task)`. If the call returns nonzero, it clears `k` and schedules another queue-processing message. If it returns zero, it does not clear `k` in that runnable; completion/state callbacks must release the in-flight state before the next file is scheduled.

The task dispatcher `b(C0363md)` chooses the Wi-Fi/P2P manager when the connection/device conditions allow and its active-count check `m.M() < 1` passes; otherwise it falls back to the basic transfer manager. This is a scheduler-level single-active-task guard for the selected Nearby manager path, not a universal claim about every unrelated download subsystem in the APK.

## 3. Native file submission granularity

Class: `com.huawei.nstackx.service.transfer.wifi.c` (log tag: `WifiDataTransferManager`).

- `c(C0363md)` dispatches by type.
- File mode enters `C(C0363md)`, performs request/response negotiation when no file client session exists, then calls `U()`.
- `U()` calls `NstackxCoreTransfer.dFileSendFilesWithRemotePath`.
- That call receives `new String[]{sourcePath}` and `new String[]{remoteId}`: one source file per native submission at this call site.
- The returned native transfer ID is registered with the session/transfer registry.

This proves single-file submission granularity in the inspected Java call site. It does not by itself prove the internal native library cannot pipeline chunks or maintain multiple internal sockets.

## 4. Other data queues

The stream queue uses its own `j` guard and `RunnableC0348jd`. The bytes queue uses `l` and `RunnableC0353kd`, with a separate retry/wait state for EAGAIN or NOT_CONNECTED. Because these queues have independent locks/flags, the Java scheduler is not a single global FIFO across all three data types. Cross-type overlap remains dependent on connection/session conditions and needs runtime measurement.

## 5. Corrected conclusions about concurrency

| Question | Finding | Confidence |
|---|---|---|
| Are selected files submitted as one multi-file array at this Java call site? | No; one-element source and remote-path arrays are passed | High |
| Does the Java file queue start multiple queued file tasks concurrently? | No; one in-flight file task is guarded by `k` | High |
| Is the entire app globally single-threaded for all transfer types? | No; three separate queues and guards exist | High |
| Can file chunks be sent concurrently inside native code? | Not determined | Unknown |
| Can different transfer types overlap in practice? | Architecturally possible; runtime conditions not measured | Unconfirmed |
| Does a completed/failed transfer advance the queue? | Scheduler is callback/state driven; exact callback mapping still needs full trace | Medium |

## 6. Implications for UltraShare

An independent UltraShare scheduler should begin with:
- A durable user-visible job queue, distinct from transient socket/session IDs.
- One active file per peer as a conservative default.
- Bounded, benchmark-driven parallelism only after measuring device CPU, memory, radio contention, and receiver storage.
- Separate control, bulk-file, and small-message lanes with explicit priorities and backpressure.
- Per-file state transitions, cancellation, bounded retry/backoff, and a persisted checkpoint protocol.
- Atomic temporary-file writes and final SHA-256 verification.

Do not copy Zapya's classes, TLV layout, protocol, or proprietary implementation.

## 7. Next required native investigation

The previous workflow collected ELF symbols and strings, but did not emit targeted machine-code disassembly. The workflow is being extended to disassemble selected exported functions from each ABI:
- KCP create/config/send/receive/update
- DFile and DMsg client/server/send entry points
- AES-GCM encrypt/decrypt
- file ACK and transfer-completion ACK helpers
- heartbeat and retry/rate-control helpers

Even targeted disassembly is only a lead. To prove a call path, follow the native control flow/call references from the JNI entry points and confirm the same path with controlled runtime traces. Symbol presence is not proof of use by every transfer mode.
