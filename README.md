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

## Reproducible analysis
Run the private GitHub Actions workflow manually or update `.github/workflows/analyze-apk.yml`. The workflow verifies the original APK SHA-256, decodes the manifest and DEX, inventories native libraries, and emits targeted native disassembly for selected transfer, KCP, ACK and AES-GCM symbols.

Analysis focuses on architecture, device discovery, connectivity, transfer reliability and background operation. Symbol presence and static decompilation are evidence leads, not proof that a component is used by every transfer mode. Runtime validation is still required.

## Independence boundary
Use findings only to design an independently implemented UltraShare engine. Do not copy Zapya source code, binaries, proprietary protocol layouts, or assets.
