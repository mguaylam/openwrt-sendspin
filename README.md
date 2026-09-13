# openwrt-sendspin

An OpenWrt package feed for [`sendspin-cli`](https://github.com/Sendspin/sendspin-cpp-cli),
the headless player for [Sendspin](https://github.com/Sendspin/spec), the
synchronized multi-room audio protocol from the Open Home Foundation used by
Music Assistant.

The goal is to turn a router with a USB DAC into a Sendspin player, and to
submit the package to [openwrt/packages](https://github.com/openwrt/packages)
once it has proven itself here.

> **Status: experimental, work in progress.** The feed does not contain a
> package yet. Nothing below has been played through a speaker.

## Status

| | |
|---|---|
| `sound/sendspin-cli` package | not written yet |
| Build against musl | verified outside OpenWrt (Alpine 3.22, ALSA only, no mDNS) |
| Build with the OpenWrt SDK | not yet — CI is in place, see below |
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

## Using the feed

Add the feed to `feeds.conf` in an OpenWrt SDK or buildroot:

```
src-git sendspin https://github.com/mguaylam/openwrt-sendspin.git
```

then:

```sh
./scripts/feeds update sendspin
./scripts/feeds install -p sendspin -a
```

Build and install instructions for the package itself will be added with the
package, once they have been run end to end.

## Continuous integration

Every push builds the feed with
[openwrt/gh-action-sdk](https://github.com/openwrt/gh-action-sdk) on the
official SDK containers, against OpenWrt 25.12.5 and the snapshot SDK:

| Target | Package arch | Why |
|---|---|---|
| `ramips/mt7621` | `mipsel_24kc` | little-endian, soft-float; the hardware under test |
| `ath79/generic` | `mips_24kc` | big-endian, soft-float |
| `mediatek/filogic` | `aarch64_cortex-a53` | 64-bit ARM |
| `x86/64` | `x86_64` | 64-bit x86 |

A green big-endian build shows the code compiles there, not that it plays
correctly.

## Dependencies

`sendspin-cli` pulls its dependencies with CMake `FetchContent` at configure
time, which the OpenWrt build system cannot allow. The package will instead:

- use the libraries already in the OpenWrt feeds: `alsa-lib` and `libopus`
  (upstream bundles its own copy of Opus through micro-opus);
- download the libraries that have no OpenWrt package as pinned, hashed source
  archives, and hand them to CMake in place of `FetchContent`:
  [sendspin-cpp](https://github.com/Sendspin/sendspin-cpp),
  [ArduinoJson](https://github.com/bblanchon/ArduinoJson),
  [IXWebSocket](https://github.com/machinezone/IXWebSocket) and
  [micro-flac](https://github.com/esphome-libs/micro-flac).

Portability fixes are proposed upstream first; patches carried here are meant
to be temporary.

## License

The packaging in this repository (Makefiles, init scripts, configuration) is
licensed under GPL-2.0-only, like openwrt/packages — see [LICENSE](LICENSE).

The packaged software keeps its own licenses: `sendspin-cli`, `sendspin-cpp`
and `micro-flac` are Apache-2.0, ArduinoJson is MIT, IXWebSocket and Opus are
BSD-3-Clause. Patches to upstream code are under the license of the code they
modify.
