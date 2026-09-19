# RNSBox — a Reticulum router for the Sipeed LicheeRV Nano-e

RNSBox turns a **Sipeed LicheeRV Nano-e** (SG2002, T-Head C906 RISC-V) into a
small, USB-C-powered [Reticulum](https://reticulum.network/) router and
transport node with an OpenWrt-style web admin UI.

It is distributed as a **patch series on top of the upstream Sipeed board
support package**, so this repository contains only the RNSBox delta — the
Linux kernel, Buildroot and the CVITEK/Sipeed BSP themselves come from
upstream and are fetched at build time.

- **Upstream base:** [`sipeed/LicheeRV-Nano-Build`](https://github.com/sipeed/LicheeRV-Nano-Build) at commit `d4003f15b`
- **The delta:** 23 patches in [`patches/`](patches) — the 22 from `main` plus one offline patch (see [Offline branch](#offline-branch)), MIT-licensed

> **You are on the `offline` branch.** It builds the same image as `main`, except
> Reticulum starts immediately at boot without waiting for an NTP time-sync (the
> board has no RTC, so the clock starts at 1970), and the shipped config carries no
> internet uplinks. See [Offline branch](#offline-branch) below. For the standard
> time-synced build, use `main`.

## Quick start

One command clones the pinned upstream base, fetches the cross toolchain,
applies the patch series and builds:

```bash
git clone https://github.com/Smit1237/rnsbox
cd rnsbox
./build.sh lite        # NCM-only image (~217 MB)
# ./build.sh dvd       # + a read-only disc of the latest Reticulum clients
```

The finished image lands under
`LicheeRV-Nano-Build/install/soc_sg2002_licheervnano_sd/images/`. Flash it with
`dd` (or a tool like balenaEtcher) to a microSD and boot the board.

> **First build is long.** It compiles a Rust host toolchain from source
> (needed for `python-cryptography` on `riscv64-musl`), so the first run takes
> roughly 40 minutes on a typical machine. Later builds reuse it.

Build host: a Linux machine set up for Buildroot (the upstream repo's
`host/ubuntu` container works). `./build.sh` needs `/usr/sbin` on `PATH` for
genimage's `mkdosfs`; it handles this itself.

## What it does

- **`rnsd`** (Reticulum 1.5.2) runs as the long-lived service: a transport node
  with a `TCPServerInterface` on `0.0.0.0:4242`, an `AutoInterface` on the
  USB-C LAN, and two public RNS-testnet uplinks preconfigured.
- **USB-C gadget = LAN.** The board presents a CDC-NCM network interface
  (`usb0 = 10.42.0.1/24`, dnsmasq DHCP + DNS). The DVD build additionally
  exposes a read-only mass-storage "disc" pre-loaded with Reticulum client apps
  for a zero-download quick start.
- **eth0 = WAN**, DHCP or static, with NAT masquerade plus per-rule port
  forwarding and open-port management.
- **`rnsbox-portal`** — a Flask admin UI on `http://10.42.0.1/` for network,
  Reticulum, WiFi and system settings — including setting the clock from your
  browser and configuring NTP servers, handy on a board with no RTC. It is the
  single source of truth for the generated `nftables` ruleset (boot and
  live-apply both call the same module).
- Optional **NomadNet** LXMF / pages node (off by default) and **AIC8800 WiFi**
  STA/AP support for the *W* board variant.
- Optional **SLIP-over-UART link to an external WiFi-HaLow (RNode) modem** — a
  [RNode_Halow_Firmware](https://github.com/I-AM-ENGINEER/RNode_Halow_Firmware)
  bridge — for long-range sub-GHz Reticulum over a 3-wire serial link, with the
  modem's own web UI reverse-proxied through the portal login. Off by default.

The design goal throughout is a minimal OS: the camera / display / audio / NPU
/ codec middleware of the stock BSP is stripped so nearly all of the 256 MB
DDR is available to Linux and the router data plane.

## Offline branch

This `offline` branch is for boxes that boot with **no internet** and shouldn't
wait for a clock they can't set. It differs from `main` by one extra patch
(`patches/0023-…`):

- **rnsd starts immediately** — `S45ntpsync` asserts the clock-ready flag at boot
  instead of waiting up to ~90 s for a default route + NTP step, so `rnsd` (and
  NomadNet, if enabled) come up right away on the board's 1970 clock.
- **`ntpd` still runs** (started with `-g`). Offline it simply retries; if the box
  ever reaches the internet it steps the clock to real time. `rnsd` tolerates that
  forward jump **without a restart** — it re-establishes links and re-learns paths
  once, then runs with correct timestamps.
- **No internet uplinks** — the two RNS-testnet `TCPClientInterface` entries are
  removed from the shipped `/etc/reticulum/config`, so an offline box makes no
  outbound connections on boot. The local `usb0` AutoInterface + TCP listener are
  unchanged, and a commented example shows how to re-add an uplink later.

Reticulum transport and routing work fine on a 1970 clock (it uses time deltas)
over the local interfaces; only a peer with a correct clock would treat our
announcements as stale, which doesn't matter on an offline mesh. LXMF / NomadNet
message timestamps read 1970 until `ntpd` corrects the clock (NomadNet is off by
default). Build it exactly like `main` — `./build.sh lite` (or `dvd`).

## HaLow modem — SLIP wiring

Wire the RNode HaLow modem to **UART1** (`/dev/ttyS1` — **not** `ttyS0`, the
serial console) with three 3.3 V-TTL lines; TX and RX cross over:

| Nano-e pad | Function | Wire to modem |
|------------|----------|---------------|
| `GPIOA28`  | UART1_TX | RX            |
| `GPIOA29`  | UART1_RX | TX            |
| `GND`      | ground   | GND           |

Set **both** ends to `1500000` baud — the Nano-e's UART tops out at 1,562,500,
below the modem's 2 Mbaud default. Enable the link from the portal (*Reticulum
tab → HaLow modem (SLIP)*), then point a `TCPClientInterface` at the modem on
**port 8001**. Full walkthrough in `README.RNSBox.md` (shipped by the patch series).

## Repository layout

```
patches/          the 23-patch RNSBox series (git am-able onto d4003f15b)
build.sh          one-command: clone upstream -> apply patches -> build
LICENSE           MIT (the RNSBox delta)
README.md         this file
```

Applying the series brings the full documentation into the built tree
(`README.RNSBox.md`, `APPLYING.md`) alongside the RNSBox sources.

## Hardware

- Sipeed LicheeRV Nano-e (no radio) or Nano-e **W** (AIC8800 WiFi). One image
  serves both; WiFi bring-up is a clean no-op where there is no radio.
- microSD: the DVD build's size tracks the current client releases (~3.3 GB
  now; use a card comfortably larger, e.g. 8 GB); the lite build is ~217 MB.
  The rootfs auto-grows to fill the card on first boot.

## Default credentials

| Service | User  | Password |
|---------|-------|----------|
| SSH     | root  | admin    |
| Web UI  | admin | admin    |

Change these on first use. The web password is stored hashed (pbkdf2) in
`/etc/rnsbox/auth.json` after first login; the Flask session key is generated
at runtime and never stored in the source tree.

## Applying the patches by hand

If you would rather drive it yourself instead of `build.sh`:

```bash
git clone https://github.com/sipeed/LicheeRV-Nano-Build.git
cd LicheeRV-Nano-Build
git checkout d4003f15b35d43ad4842f427050ab2bba0114fa5
git clone --depth=1 https://github.com/sophgo/host-tools host-tools
git am /path/to/rnsbox/patches/*.patch
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
