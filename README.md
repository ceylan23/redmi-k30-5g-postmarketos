# Redmi K30 5G → postmarketOS

Turning a **Redmi K30 5G** (`xiaomi-picasso`, Qualcomm **SM7250 / Snapdragon 765G**) into a native Linux device with **postmarketOS**, using the downstream 4.19 vendor kernel.

## Device

| | |
|---|---|
| Codename | picasso |
| SoC | Qualcomm SM7250-AB (Snapdragon 765G), Adreno 620 |
| RAM / storage | 6 GB / 128 GB (this unit) |
| Display | 6.67" 2400×1080 LCD, 120 Hz |
| Battery | 4500 mAh |
| Stock firmware | MIUI 13.0.5 stable (Android 12) — also the firmware base the LineageOS 21 picasso builds ask for |
| Bootloader | Unlocked |
| Role | Spare device, data wipe is fine |

## Why downstream kernel

There is no existing postmarketOS or Ubuntu Touch port for picasso, and SM7250 only received foundational mainline support (SoC DTSI, clocks, power domains) in Aug–Sep 2026 — there is no mainline device tree for this phone yet. The practical route is the one already proven on the sibling Redmi K30 Pro (`lmi`): postmarketOS on the downstream LineageOS / Xiaomi 4.19 kernel, where the Android drivers for panel, touch and WiFi already exist.

## Phase 1 goals

- [ ] pmaports packages: `device-xiaomi-picasso` + `linux-xiaomi-picasso` (downstream)
- [ ] Boots via `fastboot boot` first — nothing flashed until it reaches userspace
- [ ] Display + touchscreen working
- [ ] USB networking + SSH
- [ ] WiFi working
- [ ] Persistent install to `userdata` only after the above pass

Out of scope for phase 1: modem / calls / 5G, cameras, fingerprint, 120 Hz refresh.

## Layout

- `research.md` — verified porting notes (kernel sources, panel/touch/WiFi parts, boot image format, partitions, pmaports template references)
- `pmaports/` — the device and kernel packages (to be added)

## Status

2026-10-31 — **boot.img built successfully** on the owner's Arch laptop (sherbook, Ryzen 7 4700U): kernel built in one pass (~16 min), device package, `install --no-fde` and `export` all passed. See `build-summary.md`.

Artifacts (on sherbook, `~/k30-pmos/artifacts/`):
- `boot.img` — 60,174,336 bytes, sha256 `b4e40f79a6e7d6c9eb90e97310a03a7aaaa1723f135e3ff587bbd76102183cb5`, Android bootimg, page size 4096, DTB confirmed `picasso-sm7250.dtb`
- `xiaomi-picasso.img` — rootfs image, 1,363,148,800 bytes (console UI)

Not yet done: nothing has been flashed or booted on the phone. Next step is a non-destructive `fastboot boot boot.img` test.
