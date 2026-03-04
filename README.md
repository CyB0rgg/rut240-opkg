# RUT240 Community opkg Feed

Pre-built OpenWrt packages for Teltonika RUT240 routers running firmware with OpenSSL 3.0.

Teltonika's official package repository ships several packages compiled against OpenSSL 1.1 (`libopenssl1.1`) and older `libnettle`, which are incompatible with the OpenSSL 3.0 (`libopenssl3`) that ships with current RUT240 firmware (RUTX_R_00.07.x). This feed provides correctly compiled packages.

## Architecture

- **Target:** `ath79/generic`
- **Arch:** `mips_24kc`
- **Built with:** OpenWrt 23.05.5 SDK (OpenSSL 3.0, musl 1.2.4, GCC 12.3)

## Available Packages

| Package | Version | Description |
|---------|---------|-------------|
| `radsecproxy` | 1.9.1-1 | Generic RADIUS proxy for UDP/TLS (RadSec) |
| `libnettle8` | 3.9.1-1 | GNU crypto library |
| `libgmp10` | 6.2.1-1 | GNU Multiple Precision arithmetic library |

## Installation

```bash
# Add this feed to your RUT240
echo "src/gz rut240_community https://cyb0rgg.github.io/rut240-opkg/packages" \
  >> /etc/opkg/customfeeds.conf

# Install packages
opkg update
opkg install radsecproxy
```

## Building

Packages are cross-compiled using the OpenWrt 23.05.5 SDK for `ath79/generic` inside a Docker container:

```bash
docker run --platform linux/amd64 -v $(pwd)/output:/output debian:bookworm-slim
# Inside container:
apt-get update && apt-get install -y build-essential gawk unzip file wget python3 rsync ca-certificates libncurses-dev git
wget https://downloads.openwrt.org/releases/23.05.5/targets/ath79/generic/openwrt-sdk-23.05.5-ath79-generic_gcc-12.3.0_musl.Linux-x86_64.tar.xz
tar xf openwrt-sdk-*.tar.xz && cd openwrt-sdk-*
./scripts/feeds update packages base
./scripts/feeds install radsecproxy libopenssl libnettle
make defconfig
make package/feeds/packages/radsecproxy/compile V=s -j1
```

## License

Packages retain their original upstream licenses. This repository is provided as-is for the Teltonika RUT240 community.

[C] CyB0rgg @ Kiroshi.Group
