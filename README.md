# openwrt-sendspin

An OpenWrt package feed for [`sendspin-cli`](https://github.com/Sendspin/sendspin-cpp-cli),
the headless player for [Sendspin](https://github.com/Sendspin/spec), the
synchronized multi-room audio protocol from the Open Home Foundation used by
Music Assistant.

The goal is to turn a router with a USB DAC into a Sendspin player, and to
submit the package to [openwrt/packages](https://github.com/openwrt/packages)
once it has proven itself here.

> **Status: experimental.** The package runs on one router, a D-Link
> DIR-3040, playing from Music Assistant, including in multi-room groups.
> It has not been tried on other hardware yet.

## Status

| | |
|---|---|
| `sound/sendspin-cli` package (0.1.6) | written |
| Build with the OpenWrt SDK | verified for `ramips/mt7621` on 25.12.5; other targets through CI |
| Install and service on a router | verified on the DIR-3040 |
| Playback from Music Assistant, multi-room | verified on the DIR-3040 |
| Rediscovery after a service restart | verified |
| Rediscovery after a router reboot | verified, through a workaround for umdns — see [known limitations](#known-limitations) |
| USB DAC unplugged and plugged back during playback | verified, through a workaround — see [known limitations](#known-limitations) |
| Big-endian targets (e.g. ath79) | builds; playback expected to be wrong — see [known limitations](#known-limitations) |

## Hardware under test

- **D-Link DIR-3040 A1** — `ramips/mt7621`, MediaTek MT7621AT (MIPS 1004Kc,
  2 cores / 4 threads, 880 MHz, no FPU), 256 MB RAM, OpenWrt 25.12.5.
- **USB audio adapter** — generic "Yichip USB-Audio" (`12d1:3a06`), USB full
  speed, UAC1. It plays **16-bit / 48 kHz stereo only**, so that is the format
  this setup targets.

Measured on this setup while playing from Music Assistant:

| | |
|---|---|
| Format negotiated | FLAC, 48 kHz, 16-bit, stereo — no resampling on the router |
| CPU | 6.3 % of one of the four threads, decoding and software volume included |
| Memory | 5 MiB resident |
| Underruns, lost sync | none over 10 minutes and five tracks |
| Package | 246 KiB; 1.45 MiB of flash with `libopus`, `libstdcpp6` and `umdns` |

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

### Routers with more than one network

umdns only announces the player on the networks listed in
`/etc/config/umdns`, which is `lan` by default. Point it at the network the
Sendspin server reaches the router through, and make sure that network's
firewall zone accepts TCP port 8928 (the `port` option) and UDP port 5353:

```sh
uci set umdns.@umdns[0].network='<network>'
uci commit umdns
service umdns reload
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

## Known limitations

- **Big-endian targets** (e.g. `ath79`, `mips_24kc`): the package builds, but
  Opus, 16-bit FLAC and software volume are expected to play as noise, because
  a few places in the upstream code handle samples in host byte order. Found by
  reading the code, not yet reproduced. Little-endian targets are not affected.
  Tracked in [#2](https://github.com/mguaylam/openwrt-sendspin/issues/2).
- **umdns workaround.** The umdns shipped in OpenWrt 25.12 does not announce
  service instances when a network comes up, and on networks with an mDNS
  reflector it takes its own reflected probe for a name conflict and stops
  announcing. Left alone, the player is not rediscovered after a reboot. The
  package works around it with `/etc/hotplug.d/iface/50-sendspin-cli`, which
  restarts the player when a network umdns announces on comes up or changes
  address; a renewed DHCP lease with the same address does not restart it.
  Fixes are proposed upstream in
  [openwrt/mdnsd#36](https://github.com/openwrt/mdnsd/pull/36); details in
  [#5](https://github.com/mguaylam/openwrt-sendspin/issues/5).
- **USB DAC unplugged and plugged back.** sendspin-cli 0.1.6 notices the DAC
  coming back and reopens it, but then stops feeding it, and playback stutters
  in a loop of underruns until the player is restarted. The package restarts
  the player when a playback device appears
  (`/etc/hotplug.d/sound/50-sendspin-cli`). The server ends the stream on that
  restart, so playback has to be started again.
- **Client-initiated discovery**: with mDNS handled by umdns, `server` must be
  an address; `mdns:` server discovery is not available.

## TODO

- [ ] Remove the umdns workaround (`files/sendspin-cli.hotplug`) once
  [openwrt/mdnsd#36](https://github.com/openwrt/mdnsd/pull/36) ships in an
  OpenWrt release, and retest rediscovery after a reboot without it.
- [ ] Follow up on openwrt/mdnsd#36 if it has had no review by 2026-09-28,
  on the pull request or on the openwrt-devel mailing list.
- [ ] Remove the USB DAC workaround (`files/sendspin-cli.sound-hotplug`) once
  [Sendspin/sendspin-cpp-cli#54](https://github.com/Sendspin/sendspin-cpp-cli/issues/54)
  is resolved in a sendspin-cli release, and retest unplugging the DAC during
  playback without it.
- [ ] Fix playback on big-endian targets
  ([#2](https://github.com/mguaylam/openwrt-sendspin/issues/2)).

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
