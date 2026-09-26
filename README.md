# RNSBox — a Reticulum router for the Sipeed LicheeRV Nano-E

RNSBox turns a **Sipeed LicheeRV Nano-E** (SG2002, T-Head C906 RISC-V) into a
small, USB-C-powered [Reticulum](https://reticulum.network/) router and
transport node with an OpenWrt-style web admin UI.

It is distributed as a **patch series on top of the upstream Sipeed board
support package**, so this repository contains only the RNSBox delta — the
Linux kernel, Buildroot and the CVITEK/Sipeed BSP themselves come from
upstream and are fetched at build time.

- **Upstream base:** [`sipeed/LicheeRV-Nano-Build`](https://github.com/sipeed/LicheeRV-Nano-Build) at commit `d4003f15b`
- **The delta:** the patch series in [`patches/`](patches), MIT-licensed

## Quick start

One command clones the pinned upstream base, fetches the cross toolchain,
applies the patch series and builds:

```bash
git clone https://github.com/Smit1237/rnsbox
cd rnsbox
./build.sh lite        # NCM-only image (~226 MB)
# ./build.sh dvd       # + a read-only disc of the latest Reticulum clients
```

The finished image lands under
`LicheeRV-Nano-Build/install/soc_sg2002_licheervnano_sd/images/`. Flash it with
`dd` (or a tool like balenaEtcher) to a microSD, boot the board, connect it to
a computer over USB-C and open `http://10.42.0.1/`.

> **First build is long.** It compiles a Rust host toolchain from source
> (needed for `python-cryptography` on `riscv64-musl`), so the first run takes
> one to two hours. Later builds reuse it.

Build host: a Linux machine set up for Buildroot (the upstream repo's
`host/ubuntu` container works; the `dvd` build also needs `xorriso`).
`./build.sh` needs `/usr/sbin` on `PATH` for genimage's `mkdosfs`; it handles
this itself.

## What it does

- **`rnsd`** (Reticulum 1.5.2) runs as the long-lived service: a transport node
  with a `TCPServerInterface` on port 4242 (open on the WAN by default, so
  Reticulum peers can connect in), an `AutoInterface` on the LAN (the USB-C
  link and the WiFi hotspot), and two public RNS-testnet uplinks
  preconfigured.
- **Works offline.** `rnsd` starts at every boot with or without internet, NTP
  or an RTC, so HaLow-, LAN- or WiFi-only meshes work. Boot never waits for
  the time: it comes from NTP whenever a server is reachable, from your
  browser (*Settings → Sync clock to browser*), or at boot from an optional
  DS3231 RTC (wiring below). Every NTP or browser sync is written to the RTC,
  and an hourly resync keeps it on NTP time.
- **One LAN: USB-C, plus the WiFi hotspot.** The board presents a CDC-NCM
  network interface (`usb0`) over USB-C; on the *W* board the hotspot is
  bridged with it into one LAN, `br-lan = 10.42.0.1/24`, with one dnsmasq
  DHCP pool (`10.42.0.10`–`.250`) and DNS. The USB host and hotspot clients
  share the subnet and reach each other directly. `br-lan` is the only
  interface with IPv6, link-local only, for Reticulum's `AutoInterface`; a
  client's network adapter (USB or WiFi) needs IPv6 enabled (the default on
  Windows, macOS, Linux and Android), otherwise point a `TCPClientInterface`
  at `10.42.0.1:4242`. The DVD build additionally exposes a read-only
  mass-storage "disc" pre-loaded with Reticulum client apps for a
  zero-download quick start.
- **Automatic WAN**: eth0 (DHCP or static) while it has a cable, otherwise
  the WiFi client on WiFi boards (`wlan0`, DHCP), with nothing to switch by
  hand; either can be pinned. NAT masquerade plus per-rule port forwarding and
  open-port management.
- **`rnsbox-portal`** — a compact C++/CGI admin UI (served by uhttpd, ~0
  resident RAM) for network, Reticulum, WiFi, clock and system settings. Open
  it by IP address: `http://10.42.0.1/` over USB-C or the hotspot, or the
  box's WAN address. As a DNS-rebinding guard it answers only on an IP; names such
  as `setup.lan` redirect there and any other name gets 403. Every change is a
  POST from the portal's own page. It is the single source of truth for the
  generated `nftables` ruleset (boot and live-apply both call the same
  generator).
- **AIC8800 WiFi** on the *W* board: client (usable as the WAN), hotspot
  (part of the LAN, `10.42.0.0/24`, as above), or both at once. Every RNSBox
  uses that same LAN subnet, so one box can't take another's hotspot as its
  uplink; link two boxes over Reticulum through their WAN addresses.
- Optional **SLIP-over-UART link to an external WiFi-HaLow (RNode) modem** — a
  [RNode_Halow_Firmware](https://github.com/I-AM-ENGINEER/RNode_Halow_Firmware)
  bridge — for long-range sub-GHz Reticulum over a 3-wire serial link, with the
  modem's own web UI served behind the portal login on its own port, 8081.
  Off by default.
- Opt-in **NomadNet** LXMF / pages node and **LXMF propagation node**
  (`lxmd`) for store-and-forward message routing. Both ship disabled
  (`enabled=no` in `/etc/rnsbox/nomadnet.conf` / `lxmd.conf`) because RAM is
  tight alongside `rnsd`; enable them from the console.

The design goal throughout is a minimal OS: the camera / display / audio / NPU
/ codec middleware of the stock BSP is stripped so nearly all of the 256 MB
DDR is available to Linux and the router data plane.

## HaLow modem — SLIP wiring

Wire the RNode HaLow modem to **UART1** (`/dev/ttyS1` — **not** `ttyS0`, the
serial console) with three 3.3 V-TTL lines; TX and RX cross over:

| Nano pad   | Function | Wire to modem |
|------------|----------|---------------|
| `GPIOA28`  | UART1_TX | RX            |
| `GPIOA29`  | UART1_RX | TX            |
| `GND`      | ground   | GND           |

Pad positions: Sipeed's
[LicheeRV Nano pinout diagram](https://github.com/sipeed/sipeed_wiki/raw/main/docs/hardware/en/lichee/assets/RV_Nano/intro/RV_Nano_3.jpg).
Set **both** ends to `1500000` baud — the Nano-E's UART tops out at 1,562,500,
below the modem's 2 Mbaud default. Enable the link from the portal (*Reticulum
tab → HaLow modem (SLIP)*), then point a `TCPClientInterface` at the modem on
**port 8001**. Once SLIP is enabled, *Open modem web UI* on the Dashboard's
HaLow card opens the modem's own page behind the portal login, on its own port
(`http://<box>:8081/cgi-bin/portal/modem/`) so its scripts can't act on the
portal. Full walkthrough in `README.RNSBox.md` (shipped by the patch series).

## DS3231 RTC — optional wiring

The Nano-E has no battery-backed clock. A DS3231 module (e.g. the common
ZS-042 board) keeps the time across power-off. RNSBox looks for it on the
bit-banged I²C bus `i2c5` at address `0x68`:

| Nano pad   | Function | Wire to DS3231 |
|------------|----------|----------------|
| `3V3`      | power    | VCC            |
| `GND`      | ground   | GND            |
| `GPIOA15`  | I2C5 SCL | SCL            |
| `GPIOA27`  | I2C5 SDA | SDA            |

Pad positions are on Sipeed's
[LicheeRV Nano pinout diagram](https://github.com/sipeed/sipeed_wiki/raw/main/docs/hardware/en/lichee/assets/RV_Nano/intro/RV_Nano_3.jpg).
Raspberry Pi RTC modules fit too (5-pin socket `+ D C NC −` = 3V3, SDA, SCL,
unused, GND), but often lack I²C pull-ups: add 4.7 kΩ from SDA and SCL to 3V3
if `i2cdetect -y 5` doesn't show `68`.

Power it from **3V3 only**; the SG2002 pads are not 5 V tolerant. Don't use
the pads Sipeed labels I2C1 / I2C3: on the *W* board they carry the SDIO bus to
the WiFi chip. The ZS-042 trickle-charges its coin cell, so use an LIR2032 or
remove its charging resistor. An RTC that was never set or lost power is
ignored until the next NTP or browser sync writes it, and with no module fitted
the box works as before. Pull-ups and the validity rules are in
`README.RNSBox.md`.

## Repository layout

```
patches/          the RNSBox patch series (git am-able onto d4003f15b)
build.sh          one-command: clone upstream -> apply patches -> build
.github/          CI: build the lite image on GitHub Actions + publish a Release
LICENSE           MIT (the RNSBox delta)
README.md         this file
```

Applying the series brings the full documentation into the built tree
(`README.RNSBox.md`, `APPLYING.md`) alongside the RNSBox sources.

## Hardware

- Sipeed LicheeRV Nano-E (Ethernet, no radio) or a WiFi model: Nano-WE
  (Ethernet + AIC8800 WiFi) or Nano-W (AIC8800 WiFi, no Ethernet — join a
  network on the WiFi tab and it becomes the uplink).
  One image serves all of them; WiFi bring-up is a clean no-op where there is
  no radio.
- microSD: the DVD build's size tracks the current client releases (~3.3 GB
  now; use a card comfortably larger, e.g. 8 GB); the lite build is ~226 MB.
  The rootfs auto-grows to fill the card on first boot.
- Optional: a DS3231 RTC module and an RNode HaLow modem (wiring above).

## Default credentials

| Service | User  | Password |
|---------|-------|----------|
| SSH     | root  | admin    |
| Web UI  | admin | admin    |

Change both on first use, because SSH and the web UI are open on the WAN by
default. Change the web password under *Settings → Change admin password*.
Change the SSH password with `passwd` over SSH; the portal doesn't change it.
The web password is stored as a pbkdf2-sha256 hash in `/etc/rnsbox/auth.json`.
The session token is random per login and kept only in RAM.

Open on the WAN by default (*Network → Open ports*): `tcp 22` (SSH), `tcp 80`
(the web UI, plus `8081` for the modem page) and `tcp 4242` (`rnsd`, for
Reticulum peers). Remove any of them there to close it; the LAN (USB-C and
the hotspot) always has full access.

The WiFi **hotspot has no default password**: it stays off until you set one
(8–63 characters) on the WiFi tab. The old published default `changeme123` is
refused.

## Applying the patches by hand

If you would rather drive it yourself instead of `build.sh`:

```bash
git clone https://github.com/sipeed/LicheeRV-Nano-Build.git
cd LicheeRV-Nano-Build
git checkout d4003f15b35d43ad4842f427050ab2bba0114fa5
git clone --depth=1 https://github.com/sophgo/host-tools host-tools
git -c user.name=rnsbox -c user.email=rnsbox@localhost am /path/to/rnsbox/patches/*.patch   # git am commits, so it needs an identity
export PATH=/usr/sbin:/sbin:$PATH   # genimage's mkdosfs
source build/cvisetup.sh && defconfig sg2002_licheervnano_sd && build_all
./build-rnsbox.sh lite      # or: ./fetch-clients.sh && ./build-rnsbox.sh dvd
```

## Licensing

- The **RNSBox code in this patch series** (the `rnsbox-portal` app, init
  scripts, Buildroot package recipes, build scripts and configuration) is
  released under the **MIT License** — see [LICENSE](LICENSE).
- The **upstream BSP** (Linux kernel, Buildroot, Sipeed / CVITEK sources)
  remains under its own respective licenses.
- The bundled Reticulum client applications are **downloaded at build time**,
  not redistributed here. They carry their own licenses, some of which are
  non-commercial (e.g. Sideband, CC BY-NC-SA) or copyleft (e.g. Ratspeak,
  AGPL-3.0); review them before redistributing any built image.

## Support RNSBox

RNSBox is free and open source (MIT). If it's useful to you, you can support
ongoing development, test hardware, and hosting for the Reticulum testnet
uplinks with a crypto donation. The web UI also has a **Donate** page
(sidebar → Donate) with a scannable QR code for each wallet.

| Coin | Address |
| --- | --- |
| Bitcoin (BTC) | `bc1q559qfr8nlqr2s6p07hgm03x357mncydehntdmlj7hu2qaud8wkqsgycagh` |
| Ethereum (ETH) | `0x5bc9b408d67c4b8294290e1dd281526be5913864` |
| Solana (SOL) | `HjpiqhDdLd3p2ZcutYFptjX9TFiNYMWFGZUGVQnfesxM` |
| Litecoin (LTC) | `ltc1qaatf3kken6peg8z6s840w4kjad643y9gptt9775zgwkhpzuyxl3qjrpss2` |
| Dogecoin (DOGE) | `9umb1Mqvq5bYg7AG8fFghvzsFHZo9fnBmf` |
