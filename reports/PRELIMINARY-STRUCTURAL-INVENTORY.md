# Zapya APK — Preliminary Structural Inventory

## Scope
This report records facts visible in the private analysis repository's uploaded archive inventory. It is not a decompilation or behavioral analysis.

## Artifact identity
- Artifact path recorded in inventory: `apk/Zapya.apk`
- SHA-256 recorded in `inventory/SHA256.txt`:
  `0089495864823fcb50ab6eacb3795adef9efdae7ad4868246e1e8ee390cc9132`
- ZIP entry timestamps shown by the inventory: 2021-04-09 10:59.

## DEX layout
- `classes.dex`: 9,386,768 bytes
- `classes2.dex`: 9,097,380 bytes
- `classes3.dex`: 269,360 bytes

The APK uses multidex. The DEX files have not yet been disassembled, so application classes, call paths, protocol constants and transfer state machines remain unknown.

## Native libraries listed
Both ARM64 and ARMv7 variants are present for several libraries:
- `libnearby_nstackx.so` (502,392 bytes on ARM64; 378,952 bytes on ARMv7)
- `libjni.so`
- `libsecsdk.so`
- `libcore.so`
- `libdm_md5.so`
- `libwind.so`
- `libweibosdkcore.so`
- `libobserver.so` / `libobserver21.so`
- `libzbar.so`

Names alone do not prove each library's role. In particular, the presence of a library with “nearby/nstackx” in its name is a lead for investigation, not proof of the exact discovery protocol or its use by Zapya.

## Other notable archive entries
- Binary `AndroidManifest.xml`: 76,328 bytes.
- Huawei GRS route configuration JSON assets.
- FAQ/guide HTML, JavaScript, CSS and image assets.
- OpenGL shader assets for beauty/lookup/video rendering.
- Google/Firebase and Huawei service metadata files.

## Current conclusions
1. The package is a multidex Android application with native components for both ARM64 and ARMv7.
2. The archive includes a sizeable native nearby-related library; its exported symbols, strings and call sites should be examined.
3. The manifest must be decoded before making claims about permissions, services, receivers, providers, foreground execution, Wi-Fi/Bluetooth/location permissions or Android compatibility.
4. DEX analysis is required to identify discovery, connection negotiation, socket setup, transfer protocol, integrity checks, retries, queueing and background service behavior.

## Required next analysis
1. Verify the APK bytes in the private repository against the recorded SHA-256.
2. Decode the manifest and extract package/version/SDK metadata and declared components.
3. Disassemble all three DEX files and search for networking, Wi-Fi Direct, hotspot, Bluetooth, NSD/mDNS, sockets, HTTP, file I/O, checksum, retry and foreground-service code paths.
4. Inspect native library strings, exported symbols and JNI registrations.
5. Only then write a behavior map and derive an independent UltraShare design. Do not copy proprietary Zapya code or assets.

## Limitations
No decompiler output, decoded manifest, runtime traces, packet captures or native symbol analysis has been collected yet. Therefore no claim is made here about the exact way Zapya discovers peers, establishes a link, chooses a transport, or sustains transfers in the background.
