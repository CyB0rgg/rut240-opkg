<p align="center">
  <img src="https://img.shields.io/badge/ARCH-mips__24kc-ff003c?style=flat-square&labelColor=0d0d0d" />
  <img src="https://img.shields.io/badge/TARGET-ath79%2Fgeneric-00f0ff?style=flat-square&labelColor=0d0d0d" />
  <img src="https://img.shields.io/badge/SDK-OpenWrt_23.05.5-fcee0a?style=flat-square&labelColor=0d0d0d" />
  <img src="https://img.shields.io/badge/SIGNED-usign-ff003c?style=flat-square&labelColor=0d0d0d" />
</p>

```
 ██▀███   █    ██ ▄▄▄█████▓ ██▀███   ▄▄▄      ▄▄▄█████▓ ▒█████   ██▀███
▓██ ▒ ██▒ ██  ▓██▒▓  ██▒ ▓▒▓██ ▒ ██▒▒████▄    ▓  ██▒ ▓▒▒██▒  ██▒▓██ ▒ ██▒
▓██ ░▄█ ▒▓██  ▒██░▒ ▓██░ ▒░▓██ ░▄█ ▒▒██  ▀█▄  ▒ ▓██░ ▒░▒██░  ██▒▓██ ░▄█ ▒
▒██▀▀█▄  ▓▓█  ░██░░ ▓██▓ ░ ▒██▀▀█▄  ░██▄▄▄▄██ ░ ▓██▓ ░ ▒██   ██░▒██▀▀█▄
░██▓ ▒██▒▒▒█████▓   ▒██▒ ░ ░██▓ ▒██▒ ▓█   ▓██▒  ▒██▒ ░ ░ ████▓▒░░██▓ ▒██▒
░ ▒▓ ░▒▓░░▒▓▒ ▒ ▒   ▒ ░░   ░ ▒▓ ░▒▓░ ▒▒   ▓▒█░  ▒ ░░   ░ ▒░▒░▒░ ░ ▒▓ ░▒▓░
  ░▒ ░ ▒░░░▒░ ░ ░     ░      ░▒ ░ ▒░  ▒   ▒▒ ░    ░      ░ ▒ ▒░   ░▒ ░ ▒░
  ░░   ░  ░░░ ░ ░   ░        ░░   ░   ░   ▒     ░      ░ ░ ░ ▒    ░░   ░
   ░        ░                  ░           ░  ░              ░ ░     ░

              ── COMMUNITY OPKG FEED FOR TELTONIKA RUT240 ──
```

---

Pre-built OpenWrt packages for Teltonika RUT240 routers running firmware with OpenSSL 3.0.

Teltonika's official repo ships packages compiled against `libopenssl1.1` and older `libnettle` — libraries that **don't exist** on current RUT240 firmware (`RUTX_R_00.07.x`). This feed provides correctly compiled replacements built against the OpenSSL 3.0 / `libopenssl3` that actually ships on the device.

---

## // PACKAGES

| Package | Version | Notes |
|---------|---------|-------|
| `radsecproxy` | 1.11.2-1 | RADIUS proxy for UDP/TLS (RadSec). **Patched** init script — see below |
| `vim-full` | 9.0-1 | Vi IMproved — Normal build, replaces busybox `vi` |
| `libncurses6` | 6.4-2 | Terminal handling library (Unicode) |
| `terminfo` | 6.4-2 | Terminal info database (ncurses) |
| `libnettle8` | 3.9.1-1 | GNU crypto library |
| `libgmp10` | 6.2.1-1 | GNU Multiple Precision arithmetic library |

## // INSTALL

This feed is signed with [usign](https://git.openwrt.org/project/usign.git). RUT240 firmware verifies package signatures by default.

#### 1 &mdash; Add the signing key

```bash
wget -q -O /etc/opkg/keys/99d9bdd209118a07 \
  https://raw.githubusercontent.com/CyB0rgg/rut240-opkg/main/signing.pub
```

#### 2 &mdash; Add the feed

```bash
echo "src/gz rut240_community https://cyb0rgg.github.io/rut240-opkg/packages" \
  >> /etc/opkg/customfeeds.conf
```

#### 3 &mdash; Install

```bash
opkg update

# radsecproxy (RadSec/RADIUS proxy)
opkg install radsecproxy --force-depends --force-overwrite

# vim-full (replaces busybox vi)
opkg install vim-full --force-depends
```

> **Why `--force-depends`?** Teltonika's repo ships broken versions of both `radsecproxy` and `vim-full` compiled against libraries that don't exist on current firmware (`libopenssl1.1`, wrong `libncurses6`). When opkg sees these broken packages alongside ours, it gets confused during dependency resolution and reports "incompatible architectures" even though both are `mips_24kc`. `--force-depends` tells opkg to skip the broken candidates and install ours. `--force-overwrite` (radsecproxy only) handles a shared library conflict with Teltonika's pre-installed `libmosquitto-ssl`.

## // PACKAGE NOTES

### radsecproxy

The stock OpenWrt init script for radsecproxy does **not** include `procd_set_param stdout` / `procd_set_param stderr` directives. radsecproxy 1.11.x in foreground mode (`-f`, required for procd) writes all log output to **stderr** — even when `LogDestination x-syslog:///` is set in the config. Without the procd stderr routing, logs silently go to `/dev/null` and `logread` shows nothing.

This package patches the init script to route both stdout and stderr through procd to `logd`:

```
procd_set_param stdout 1
procd_set_param stderr 1
```

After installing, logs are visible via `logread | grep radsecproxy`.

### vim-full

vim 9.0 will print `E1187: Failed to source defaults.vim` on startup if no `~/.vimrc` exists. The `vim-runtime` package (2.2 MB) is intentionally not included in this feed to save flash space. Create any `.vimrc` to suppress the error:

```bash
touch ~/.vimrc
```

## // BUILD

Cross-compiled using the OpenWrt 23.05.5 SDK inside a Docker container:

```bash
docker run --platform linux/amd64 -v $(pwd)/output:/output debian:bookworm-slim
# Inside container:
apt-get update && apt-get install -y \
  build-essential gawk unzip file wget python3 rsync ca-certificates libncurses-dev git
wget https://downloads.openwrt.org/releases/23.05.5/targets/ath79/generic/\
openwrt-sdk-23.05.5-ath79-generic_gcc-12.3.0_musl.Linux-x86_64.tar.xz
tar xf openwrt-sdk-*.tar.xz && cd openwrt-sdk-*
./scripts/feeds update packages base
./scripts/feeds install radsecproxy libopenssl libnettle
make defconfig
make package/feeds/packages/radsecproxy/compile V=s -j1
```

To build a newer version, edit `PKG_VERSION` and `PKG_HASH` in the feed Makefile and remove any stale `patches/` that don't apply.

## // LICENSE

Packages retain their original upstream licenses. This repository is provided as-is for the Teltonika RUT240 community.

---

<p align="center">
  <img src="https://img.shields.io/badge/CyB0rgg-Kiroshi.Group-fcee0a?style=flat-square&labelColor=0d0d0d" />
</p>
