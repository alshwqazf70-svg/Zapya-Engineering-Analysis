# Zapya Engineering Analysis

Private research workspace for understanding APK architecture and designing an independent UltraShare transfer engine.

## Contents
- apk/: Original APK supplied for private analysis.
- inventory/: APK hash and archive inventory.
- reports/: Engineering findings and analysis reports.

## Analysis phases
- Phase 1: Static architecture, discovery, transport candidates and native symbol inventory.
- Phase 2: Control negotiation and receive-path notes.
- Phase 3: Native reliability/crypto evidence and unresolved questions.
- Phase 4: Application task model and remaining unknowns.
- Phase 5: Java scheduler call-chain findings, including per-file submission and queue concurrency.
- Phase 6: ARM64/ARMv7 native call-chain evidence separating AES-GCM file frames and file ACK/retransmission from the conditional KCP DMsg path.

## Reproducible analysis
Run the private GitHub Actions workflow manually or update `.github/workflows/analyze-apk.yml`. The workflow verifies the original APK SHA-256, decodes the manifest and DEX, inventories native libraries, and emits ABI-separated native disassembly (AArch64 and Thumb where present) for file-frame encryption, file ACK/retransmission, DMsg/KCP and AES-GCM call chains.

Analysis focuses on architecture, device discovery, connectivity, transfer reliability and background operation. Symbol presence and static decompilation are evidence leads, not proof that a component is used by every transfer mode. Phase 6 evidence shows AES-GCM calls in the file-frame builder/receiver and KCP calls in the DMsg path; it does not prove KCP is used for file payloads. Runtime validation is still required. See `reports/PHASE-6-NATIVE-FILE-VS-KCP-CALLCHAIN.md`.

## Independence boundary
Use findings only to design an independently implemented UltraShare engine. Do not copy Zapya source code, binaries, proprietary protocol layouts, or assets.
