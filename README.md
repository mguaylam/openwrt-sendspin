# openwrt-sendspin

An OpenWrt package feed for [`sendspin-cli`](https://github.com/Sendspin/sendspin-cpp-cli),
the headless player for [Sendspin](https://github.com/Sendspin/spec), the
synchronized multi-room audio protocol from the Open Home Foundation used by
Music Assistant.

The goal is to turn a router with a USB DAC into a Sendspin player, and to
submit the package to [openwrt/packages](https://github.com/openwrt/packages)
once it has proven itself here.

> **Status: experimental, work in progress.** The package builds, but it has
> not been installed or played through a speaker yet.

## Status

| | |
|---|---|
| `sound/sendspin-cli` package (0.1.6) | written |
| Build with the OpenWrt SDK | verified for `ramips/mt7621` on 25.12.5; other targets through CI |
| Install and service on a router | not yet |
| Playback on hardware | not yet |
| Big-endian targets (e.g. ath79) | expected to build; playback is known to be wrong upstream for Opus, 16-bit FLAC and software volume |

## Hardware under test

- **D-Link DIR-3040 A1** — `ramips/mt7621`, MediaTek MT7621AT (MIPS 1004Kc,
  2 cores / 4 threads, 880 MHz, no FPU), 256 MB RAM, OpenWrt 25.12.5.
- **USB audio adapter** — generic "Yichip USB-Audio" (`12d1:3a06`), USB full
  speed, UAC1. It plays **16-bit / 48 kHz stereo only**, so that is the format
  this setup targets.

Reports from other hardware are welcome; please include the output of
`cat /proc/asound/card*/stream*`.

## Building

### With the SDK container, as CI does

This uses [openwrt/gh-action-sdk](https://github.com/openwrt/gh-action-sdk)
and the official SDK images, with Podman. From a directory holding a clone of
this repository:

```sh
git clone https://github.com/openwrt/gh-action-sdk.git
podman build --build-arg ARCH=ramips-mt7621-25.12.5 -t openwrt-sdk-mt7621 gh-action-sdk
mkdir -p out
podman run --rm --userns=keep-id \
  -e PACKAGES=sendspin-cli -e FEEDNAME=sendspin \
  -v "$PWD/openwrt-sendspin:/feed:Z" -v "$PWD/out:/artifacts:Z" \
  openwrt-sdk-mt7621
```

The package lands in `out/bin/packages/mipsel_24kc/sendspin/`. Other targets
use the matching image tag, e.g. `ath79-generic-25.12.5`,
`mediatek-filogic-25.12.5` or `x86-64-25.12.5`.

### In an existing SDK or buildroot

Add the feed to `feeds.conf`:

```
src-git sendspin https://github.com/mguaylam/openwrt-sendspin.git
```

then:

```sh
./scripts/feeds update sendspin
./scripts/feeds install -p sendspin sendspin-cli
make package/sendspin-cli/compile
```

## Configuration

> Not yet verified on a router.

The player is configured in `/etc/config/sendspin-cli` and disabled until
`enabled` is set. The defaults target the first sound card (`hw:0,0`) and
offer every format it accepts; `sendspin-cli -l` lists the devices and their
formats.

```sh
uci set sendspin-cli.main.enabled='1'
uci commit sendspin-cli
service sendspin-cli start
```

A Sendspin server then finds the player over mDNS. Setting `server` makes the
player connect out to that server instead. The running player can be driven
locally:

```sh
sendspin-cli status --control-socket /var/run/sendspin-cli/main.sock
```

## Design notes

- **No network at build time, no patches.** `sendspin-cli` pulls its
  dependencies with CMake `FetchContent`. The package downloads them as pinned,
  hashed source archives instead —
  [sendspin-cpp](https://github.com/Sendspin/sendspin-cpp),
  [ArduinoJson](https://github.com/bblanchon/ArduinoJson),
  [IXWebSocket](https://github.com/machinezone/IXWebSocket) and
  [micro-flac](https://github.com/esphome-libs/micro-flac) — and hands them to
  CMake with `FETCHCONTENT_SOURCE_DIR_*` and `FETCHCONTENT_FULLY_DISCONNECTED`.
- **Feed libraries where they exist.** `alsa-lib` and `libopus` come from the
  OpenWrt feeds. A small CMake shim stands in for micro-opus, the bundled copy
  of Opus upstream would otherwise build.
- **mDNS through umdns.** The player is built without mDNS; its procd service
  announces `_sendspin._tcp` through umdns instead, which avoids Avahi and D-Bus.
- **Unprivileged.** The service runs as the `sendspin` user in the `audio`
  group. State (volume, static delay, last server) is kept in RAM by default to
  spare the flash.
- **No MIPS16.** Audio decoding runs on the CPU in real time, so the package is
  built without MIPS16, like mpd and pulseaudio.

## Continuous integration

Every push builds the feed with gh-action-sdk against OpenWrt 25.12.5 and the
snapshot SDK:

| Target | Package arch | Why |
|---|---|---|
| `ramips/mt7621` | `mipsel_24kc` | little-endian, soft-float; the hardware under test |
| `ath79/generic` | `mips_24kc` | big-endian, soft-float |
| `mediatek/filogic` | `aarch64_cortex-a53` | 64-bit ARM |
| `x86/64` | `x86_64` | 64-bit x86 |

A green big-endian build shows the code compiles there, not that it plays
correctly.

## License

The packaging in this repository (Makefiles, init scripts, configuration) is
licensed under GPL-2.0-only, like openwrt/packages — see [LICENSE](LICENSE).

The packaged software keeps its own licenses: `sendspin-cli`, `sendspin-cpp`
and `micro-flac` are Apache-2.0, ArduinoJson is MIT, IXWebSocket and Opus are
BSD-3-Clause.
