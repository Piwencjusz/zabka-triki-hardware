# Żabka Triki — Firmware / OTA Notes

Companion to the hardware notes in this repo. The README covers the physical side (SWD pads, APPROTECT, the external MX25R8035F). This file is about the **firmware itself** — where it lives, how to get a copy, and why that copy isn't as useful as you'd hope.

**TL;DR:** the nRF52810 is APPROTECT-locked, so you can't read the firmware over SWD. But the Żappka app updates the cap over the air, so the image has to come from *somewhere* — and it does, off a Żabka CDN. You can pull the exact signed firmware package the app flashes, no chip access needed. The catch: **the application image is encrypted**, and the key sits in the on-chip bootloader, so the downloaded copy can't be decrypted off-device.

---

## The app doesn't ship the firmware

The Żappka app is a **Flutter** app. Unpack `base.apk` and there's no firmware blob anywhere — no `.bin`, no DFU `.zip`, nothing in `assets/` or `res/raw/`. What *is* there:

- the full **Nordic Secure DFU** stack + the `nordic_dfu` Flutter plugin (`dev.steenbakker.nordic_dfu`)
- a DFU notification icon (`ic_stat_notify_dfu.png`)
- …but no image to flash.

So the app can update the cap, but it fetches the firmware at runtime. The interesting logic isn't in the Java/Kotlin `dex` either — it's Dart, compiled into `libapp.so`, which lives in the **arm64 split** (`split_config.arm64_v8a.apk`), not in `base.apk`.

Pulling strings out of `libapp.so` gives up the whole update mechanism, including a leaked CI path confirming the source project: `capsapps-zappka-flutter-module`.

---

## The OTA endpoints

Everything is served off one host, unauthenticated (plain `GET`, no token):

| Thing | URL |
| --- | --- |
| Host | `https://game-sdk-assets.spapp.zabka.pl` |
| Manifest | `/remote-assets/manifests/manifest_1.3.3.json` |
| Assets | `/remote-assets/assets/<sha256>` |

Assets are **content-addressed**: the manifest lists each firmware entry with its **SHA-256**, and you download the file from `/remote-assets/assets/<that-hash>`. The hash is both the asset ID and the download integrity check. Example of a real asset:

```
https://game-sdk-assets.spapp.zabka.pl/remote-assets/assets/28ce70673e65691b291a88889bd043b9edb299c911a3fc05dcdf64b1435af535
```

There are **two firmware images** per manifest — `firmwareAssetEntryA` and `firmwareAssetEntryB` (peer A / peer B). The app reads the cap's current firmware version over a BLE characteristic (`_firmwareVersionStringCharUuid`) and picks one ("*defaulting to peer A first*"). Grab both.

### Reproduce

```bash
# 1. manifest -> firmware entries + their SHA-256 hashes
curl -s https://game-sdk-assets.spapp.zabka.pl/remote-assets/manifests/manifest_1.3.3.json | jq .

# 2. download an asset by its hash
curl -O https://game-sdk-assets.spapp.zabka.pl/remote-assets/assets/<sha256>

# 3. it's a Nordic DFU zip
unzip <downloaded> -d dfu/ && cat dfu/manifest.json
```

Older manifests (`manifest_1.0.0.json`, `1.1.0`, …) may exist and might predate the encryption — worth a try.

---

## What's in the package

Standard Nordic DFU layout:

```
firmware.zip
├── manifest.json          # points to the .bin and .dat
├── nrf52810_xxaa.bin      # the firmware image (application)
└── nrf52810_xxaa.dat      # init packet (signed metadata)
```

`manifest.json` only has an `application` section — **no SoftDevice, no bootloader** in the package.

### `.dat` init packet (decoded)

The `.dat` is a signed protobuf (`dfu-cc.proto`). Decoded:

```
InitCommand
  fw_version = 0x03000104
  hw_version = 52
  sd_req     = 0x00B8        # required SoftDevice FWID (nRF52810 -> an S112 build)
  type       = APPLICATION
  app_size   = 53900         # == .bin size
  hash       = SHA-256: 219a263f...ade83f6
  field 10   = c26f9749e02dd6cba479fa29   # 12 bytes -> AES-GCM nonce
signature_type = ECDSA P-256 / SHA-256
signature      = c8069ab0...  (64 bytes)
```

Useful bits:
- `sd_req = 0xB8` tells you which SoftDevice the app expects — look it up in Nordic's FWID table. Its size sets the **application base address** (i.e. the Ghidra load offset — *not* `0x0`).
- The whole thing is **ECDSA P-256 signed**, so you can't forge a valid update without Żabka's private key.
- That stray **12-byte** field is nonce-shaped → the image is very likely **AES-GCM** encrypted (see below).

---

## The catch: the image is encrypted

`nrf52810_xxaa.bin` is **not** plaintext ARM:

| Check | Result |
| --- | --- |
| Size | 53,900 bytes (`0xD28C`) |
| Shannon entropy | **7.997 / 8.0** (random-looking) |
| `0x00` / `0xFF` share | 0.4% / 0.4% |
| Vector table | garbage — initial SP reads `0x6827CD4F` (should be `0x2000xxxx`) |

Near-max entropy + a nonsense vector table + that 12-byte nonce in the init packet = the image is encrypted, almost certainly **AES-GCM**. (Inference — we can't see the bootloader's cipher call, so treat "AES-GCM" as a strong guess, not a confirmed fact.)

**Where's the key?** Not in the app. `libapp.so` has no Dart crypto libraries, no AES/GCM logic in the firmware path, no key constants — and it just hands the downloaded bytes to the stock `nordic_dfu` plugin unchanged. The other native libs don't do it either. So decryption happens in a **custom bootloader on the nRF52810**, and the AES key lives in that bootloader — inside the APPROTECT-locked flash.

Consequences:
- The downloaded image **can't be decrypted off-device**. Brute-forcing AES is not a thing.
- On the chip, though, the app is stored **plaintext** — nRF52 runs code XIP from flash, no on-the-fly decryption. The bootloader decrypts once during the update and writes plaintext. So the encryption only protects the **OTA channel**; a chip dump would still yield plaintext.
- This also means the external `MX25R8035F` isn't where the app code lives — that's game assets / data. The firmware is in the nRF52810 internal flash (locked).

---

## Can you just flash it back after `nrf52_recover`?

Not usefully:

1. It's **encrypted** — flashing the ciphertext over SWD (no bootloader in the path to decrypt) = a brick.
2. It's **application-only** — no SoftDevice, no bootloader. Even plaintext, it wouldn't run on a blank chip without the matching S112 at `0x0`.
3. `nrf52_recover` wipes the original bootloader + its AES key + SoftDevice, which are locked and unreadable — once erased, gone for good.

For **restoring** the cap you don't need any of this: just re-flash the downloaded signed package through the **normal DFU path** and the on-device bootloader decrypts + places it. Handy safety net if you brick your own firmware experiments.

---

## Status / takeaways

- ✅ Firmware is OTA, not bundled — full distribution path recovered (host / manifest / hash-addressed assets).
- ✅ Real signed DFU package pulled (peer A + B), metadata decoded (version, `sd_req` 0xB8, sizes, hash, signature).
- ⛔ Image is **encrypted (AES-GCM, likely)**; key is in the locked bootloader → no off-device decryption.
- ➡️ Plaintext firmware still only comes from a **chip dump** (hardened APPROTECT — research territory, no public rev-3 glitch).
- ➡️ For the **BLE protocol** (NUS command set, IMU packet format) you don't need the firmware at all — run **blutter** on `libapp.so`; the Dart `device_command_service` has it.

---

*Educational / own-device research, same as the rest of this repo.*
