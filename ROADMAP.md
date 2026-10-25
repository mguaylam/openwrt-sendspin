# Roadmap: which routers this package can serve

The goal of this feed is a Sendspin player on an ordinary router with a USB
DAC. This page tracks how far that reaches today, what stops it reaching
further, and which upstream change would lift each limit — so that when one
lands, it is obvious what it unblocks here.

The [README](README.md) holds the tactical TODO. This page holds the coverage
story.

## What "covered" means

Three things, in order. A target is only covered when all three hold:

1. **The package builds** for that architecture.
2. **The audio is correct** — not merely that the player starts.
3. **The server finds the player again** after a reboot, without a hand on it.

Point 2 is why big-endian is not covered, and point 3 is why no target is
fully covered yet.

## Architecture coverage

| Family | Package arch | Endianness | Status |
|---|---|---|---|
| `ramips/mt7621` | `mipsel_24kc` | little | **verified on hardware** — the DIR-3040 this was built on |
| `mediatek/filogic` | `aarch64_cortex-a53` | little | builds in CI; not run on hardware |
| `x86/64` | `x86_64` | little | builds in CI; not run on hardware |
| Other little-endian (`ipq40xx`, `mvebu`, `sunxi`, `rockchip`, `bcm27xx`, `qualcommax`…) | various | little | expected to build; untested |
| `ath79` | `mips_24kc` | **big** | **not offered** — blocked by byte order |
| `lantiq`, `realtek` | `mips_*` | **big** | **not offered** — same |
| `bmips` | `mips_mips32` | **big** | **not offered** — same |
| `mpc85xx` | `powerpc_8548` | **big** | **not offered** — same |
| `octeon` | `mips64_octeonplus` | **big** | **not offered** — same |

Little-endian is most of OpenWrt and all of its recent hardware. Big-endian is
a real slice all the same: six targets, five of them still on kernel 6.18.
Among those, `ath79` is the one that matters for this package — it is the
classic older router with a USB port, which is exactly this project's use
case. `realtek` is switch silicon, `bmips` is DSL modems, and `mpc85xx` and
`octeon` are small families.

## Requirements beyond the architecture

These are device properties, not target properties, and they rule out plenty
of otherwise supported hardware:

| | |
|---|---|
| **A USB port** | The player outputs through ALSA to a USB DAC. No I²S support. |
| **Flash** | About 1.45 MiB installed, with `libopus`, `libstdcpp6` and `umdns`, measured on 0.3.0; the 0.4.0 package is 127 KiB larger. Comfortable on 16 MB, tight on 8 MB, out of the question on 4 MB. |
| **RAM** | 5 MiB resident measured. Fine on 64 MB. |
| **CPU** | 6.3 % of one thread on an 880 MHz MIPS 1004Kc without FPU, decoding FLAC 48 kHz/16-bit with software volume. Comparable single-core hardware should cope. |

## Blockers, and what lifts them

### 1. Byte order — blocks every big-endian target

Audio samples are written in host byte order in three places and consumed as
little-endian PCM. **All three are reproduced** under `qemu-mips-static`.
Until they are fixed, the package carries `@!BIG_ENDIAN` and is simply not
offered rather than offered and wrong.

| Upstream | What | Evidence | Status |
|---|---|---|---|
| [sendspin-cpp-cli#70](https://github.com/Sendspin/sendspin-cpp-cli/issues/70) | `apply_volume()` casts the buffer at native width for 16- and 32-bit | **reproduced**: 57/64 and 60/64 samples wrong on MIPS BE, exact on x86_64 | fixed in [#77](https://github.com/Sendspin/sendspin-cpp-cli/pull/77), released in 0.4.0 |
| [micro-flac#36](https://github.com/esphome-libs/micro-flac/issues/36) | `write_samples()` fast paths cast; the generic and 24-bit ones are byte-wise | **reproduced**, and gated on buffer alignment — correct or corrupt by address | fixed in [#54](https://github.com/esphome-libs/micro-flac/pull/54), released in v0.3.0 |
| [sendspin-cpp#132](https://github.com/Sendspin/sendspin-cpp/issues/132) | `opus_decode()` fills its output with native `opus_int16` | **reproduced**: the decoder writes -45, a little-endian reader gets -11265 | fixed in [#200](https://github.com/Sendspin/sendspin-cpp/pull/200), released in v0.9.4 |

Each was demonstrated by building the file or library unchanged for MIPS
big-endian and running it under `qemu-mips-static`, with the same harness on
x86_64 as the control. The pattern is the same in all three: the careful path
writes bytes one at a time and is correct anywhere, the convenient path casts
the buffer at native width.

sendspin-cli fixed the first on 2026-10-07 in sendspin-cpp-cli#77: both paths
now assemble little-endian bytes, a new test builds its samples as bytes, and
a CI job runs it under `qemu-mips-static`, so the defect cannot come back
unnoticed. It shipped in sendspin-cli 0.4.0. Its audit found one more site, the
PortAudio sink (sendspin-cpp-cli#78), which this package does not build.

micro-flac fixed the second on 2026-10-11 in micro-flac#54: a new
`FLAC_LITTLE_ENDIAN` macro keeps the native-width fast paths to little-endian
targets, and big-endian ones take the byte-wise code, so packed PCM is
little-endian on every host. The `int32_t*` overload of `decode()` still
returns host byte order, but sendspin-cpp calls the `uint8_t*` one, which is
the one fixed. Upstream verified it under s390x with QEMU, and little-endian
code generation is unchanged. It shipped in micro-flac v0.3.0 the next day.

sendspin-cpp fixed the third on 2026-10-12 in sendspin-cpp#200: the samples
are byte-swapped after `opus_decode()`, and before `opus_encode()` on the
source side, behind `if constexpr (std::endian::native == std::endian::big)`,
so little-endian builds compile it to nothing. The same pull request moves the
host build to micro-flac v0.3.0. Upstream has not run it on a big-endian
target and asked for a test there. Both fixes shipped in sendspin-cpp v0.9.4
the same day; ESP targets stay on micro-flac 0.1.1 until ESPHome moves, which
does not matter on little-endian hardware.

sendspin-cli 0.4.0, released on 2026-10-15 and packaged here since, carries
all three: its own volume fix, and sendspin-cpp v0.9.5 with micro-flac v0.3.0.
What is left is to check the package on big-endian before the guard comes
off, since upstream has not played audio on such a target.

Tracked here as [#2](https://github.com/mguaylam/openwrt-sendspin/issues/2).

**Once checked on big-endian:** drop `@!BIG_ENDIAN` from `DEPENDS`, and `ath79`,
`lantiq`, `realtek`, `mpc85xx`, `octeon` and `bmips` become available in one
line. The `ath79` runtime test comes back with them.

### 2. Rediscovery after a reboot — affects every target

umdns does not announce service instances when a network comes up, and on a
network with an mDNS reflector it mistakes its own reflected probe for a name
conflict and stops announcing. The package works around it by restarting the
player on `ifup`, which is a race — and one that has been observed losing.

| Upstream | What | Status |
|---|---|---|
| [openwrt/mdnsd#36](https://github.com/openwrt/mdnsd/pull/36) | Both fixes | Open since 2026-09-14, no review. The GitHub repo is a mirror; the canonical tree is `git.openwrt.org`, which saw 5 commits in all of 2026. |

Tracked here as [#5](https://github.com/mguaylam/openwrt-sendspin/issues/5).

**When it lands in an OpenWrt release:** delete
`files/sendspin-cli.hotplug`, retest rediscovery after a reboot without it.

### 3. Runtime test on big-endian — affects confidence, not coverage

CI runs the package in an `x86_64` `openwrt/rootfs` image, as openwrt/packages
does. It used to run on `ath79/generic` as well, which is big-endian and where
the package is no longer built. That test returns with blocker 1.

### 4. Hardware volume — a quality limit, not a coverage one

The player attenuates in software, so every dB of volume reduction costs
resolution on a 16-bit output. A USB DAC with a hardware mixer — which is
common — could do it losslessly, but driving one is upstream roadmap item 15
(`-V`, `snd_mixer_*`), not implemented in 0.4.0. A `mixer` UCI option here
depends on it.

### 5. Encrypted transport — done in 0.4.0

The Sendspin spec authenticates in the handshake with a Noise exchange and a
PSK. Up to 0.3.0 the player sent the legacy clear-text `client/hello`, which a
server accepts only while it runs in transition mode.

| Upstream | What | Status |
|---|---|---|
| [sendspin-cpp#140](https://github.com/Sendspin/sendspin-cpp/pull/140) | Sendspin `1.0.0-rc1`: Noise KKpsk2, pairing, RC1 `client/state` | Released in sendspin-cpp v0.9.0 on 2026-10-07, and in sendspin-cli 0.4.0 on 2026-10-15. |
| [sendspin-cpp#131](https://github.com/Sendspin/sendspin-cpp/issues/131) | `client/state` flagged non-compliant: legacy `state`, missing timing fields | Closed as superseded by #140. |

Since 0.4.0 every connection is encrypted, and a server must pair with the
player before it can play. The `client_id` is the player's X25519 public key,
generated on first start, so the package now keeps the state file in
`/etc/sendspin-cli` and through a sysupgrade: in RAM, every reboot would make a
new player that each server would have to pair with again.

The package pairs from the shell with `/etc/init.d/sendspin-cli pair`, which
prints the token to paste into the server and the code of an attempt in
progress, and answers a waiting attempt with `pair confirm` or `pair cancel`.
The LuCI app does the same from *Services → Sendspin*. `allow_unpaired` lets
any server play, and is off by default.

An rc1 player cannot talk to an older server. Music Assistant speaks rc1 since
2.10.7, released on 2026-10-17 with aiosendspin 10.2.0, so the bump waited for
that. An rc1 server still accepts older players, so the order of upgrades does
not matter on that side.

## Submission to openwrt/packages

Submitted as
[openwrt/packages#30577](https://github.com/openwrt/packages/pull/30577), with
`@!BIG_ENDIAN` in place rather than waiting for blocker 1 to lift. The
fixes have now reached this package, but they are not yet checked on a
big-endian target, and
waiting would mean withholding a package that is correct on every
little-endian target — which is most of OpenWrt and all of its recent hardware
— for the sake of six targets it would be wrong on. Declining to build there
is the honest way to say that, and the guard comes off in one line when the
fixes land.

The mechanical requirements are met — maintainer, SPDX licence, procd init,
`conffiles`, no patches, no out-of-tree dependencies — and were checked
against the repository's own
[review rules](https://github.com/openwrt/packages/blob/master/.github/llm-review-rules.md).
Two points that looked like findings and are not:

- **`test-version.sh` is not needed.** The generic check runs the binary with
  `--version` and expects `PKG_VERSION` in the output; `sendspin-cli
  --version` prints `sendspin-cli 0.4.0`.
- **`codeload.github.com` is correct here, not a missing `@GITHUB`.** That
  macro resolves to `https://raw.githubusercontent.com` in
  `scripts/projectsmirrors.json`, which serves individual files and cannot
  serve a release tarball.

What is still owed is time on hardware: rediscovery after a reboot is a known
race (blocker 2), and how often it actually loses is measured in the
[README](README.md)'s TODO, not yet answered.
