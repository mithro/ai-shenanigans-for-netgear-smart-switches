# Netgear Plus Switch Internals: Complete Resource Guide

A curated collection of teardowns, reverse engineering projects, security advisories, datasheets, community tools, and forum threads documenting the hardware and firmware internals of Netgear's low-end "Plus" and "Smart Managed Plus" switches.

---

## Reverse Engineering & Firmware Analysis

### florolf's BCM53128 / GS108 Deep Dive (2019)

The single most detailed public analysis of the Plus switch firmware architecture. Florolf dumped the SPI flash from a GS108 v4, identified it as 8051 machine code running on the BCM53128's integrated microcontroller, wrote custom 8051 code to extract the mask-programmed internal ROM via GPIO bit-banging, and documented the hidden CLI and RTOS primitives inside the ROM.

- **Blog post:** [VLANs (and more shenanigans) on the Netgear GS108 "dumb" switch](https://blog.n621.de/2019/04/vlans-on-the-netgear-gs108-switch/)
- **GitHub — neatgear:** [github.com/florolf/neatgear](https://github.com/florolf/neatgear) — Tool to patch SPI flash dumps and add 802.1Q VLAN support to the unmanaged GS108. Includes 8051 disassembly notes and ROM extraction tooling.

### GS108T v2 Reverse Engineering (fvollmer)

A higher-tier "Smart Managed Pro" switch but architecturally instructive. Documents UART access (9600 baud, hidden under glued-on heatsink), CFE bootloader interruption, eCos RTOS source builds, and a partial OpenWrt port attempt. Useful for understanding where the Plus line diverges from the Pro line.

- **GitHub:** [github.com/fvollmer/GS108Tv2-reverse-engineering](https://github.com/fvollmer/GS108Tv2-reverse-engineering)
- **Related issue — GS110TP similarities:** [github.com/fvollmer/GS108Tv2-reverse-engineering/issues/1](https://github.com/fvollmer/GS108Tv2-reverse-engineering/issues/1)

### GS308E v4 / RTL8370N Teardown (OpenWrt Forum, 2025)

A user opened a GS308E v4, identified the Realtek RTL8370N switch chip and Winbond W25Q32 (4 MB) SPI flash, found a UART interface running at 56000 baud 8N1, and dumped the firmware. Boot menu output was visible on UART but no useful interactive shell was available.

- **OpenWrt forum thread:** [Netgear GS308E v4, adding support for RTL8370N based switches](https://forum.openwrt.org/t/netgear-gs308e-v4-adding-support-for-rtl8370n-based-switches/221816)

### Switcher Project (Hackaday.io)

An alternative approach: rather than replacing firmware, this project attaches a USB interface module to unmanaged Netgear GS108 switches and uses the Linux kernel's DSA (Distributed Switch Architecture) framework with existing B53 and RTL8365MB drivers to manage the switch externally from a host computer.

- **Hackaday.io project page:** [Switcher: Managing unmanaged switches](https://hackaday.io/project/182155-switcher-managing-unmanaged-switches)

---

## Hardware Identification & Teardowns

### TechInfoDepot Wiki Pages

Detailed per-model hardware inventories including chip IDs, FCC data (where applicable), flash sizes, and board photos.

- [Netgear GS105E v1](https://techinfodepot.shoutwiki.com/wiki/Netgear_GS105E_v1) — BCM53114 + Taifatech TF-480-ACL
- [Netgear GS108PE v1](https://techinfodepot.shoutwiki.com/wiki/Netgear_GS108PE_v1) — BCM53118 + Taifatech TF-480
- [Netgear GS108PE v3](https://techinfodepot.shoutwiki.com/wiki/Netgear_GS108PE_v3) — BCM53128 (integrated 8051)
- [Netgear GS108E v3](https://techinfodepot.shoutwiki.com/wiki/Netgear_GS108E_v3) — Broadcom BCM531xx
- [Netgear GS110EMX](https://techinfodepot.shoutwiki.com/wiki/Netgear_GS110EMX) — Marvell 88E6390X + 88F6811 + 88X3310P

### WikiDevi / wi-cat Database

Comprehensive cross-reference of the entire Netgear GS series with chip identifications, board revisions, and hardware version tracking.

- [Netgear GS series master list](https://wikidevi.wi-cat.ru/Netgear_GS_series)

### ServeTheHome Review (GS110EMX)

Professional review with internal photos showing the Marvell three-chip architecture (88F6811 ARM CPU + 88E6390X switch ASIC + 88X3310P 10G PHY), Nanya DDR3 RAM, and dual Macronix SPI flash chips.

- [Netgear GS110EMX Review — A Managed GS110MX Switch](https://www.servethehome.com/netgear-gs110emx-review-a-managed-gs110mx-switch/)

### FCC Filings

Note: wired-only Ethernet switches do **not** have FCC ID filings (no intentional radio transmitter). They comply via Declaration of Conformity under Part 15 Subpart B. Netgear's FCC grantee code PY3 covers only wireless products. This page is listed here to save others the dead-end search.

- [Netgear FCC ID applications (PY3)](https://fccid.io/PY3)

---

## Switch Silicon Datasheets & Documentation

### Broadcom RoboSwitch Family

The BCM5311x/BCM5312x "RoboSwitch" chips power the majority of older Plus switches. They integrate gigabit PHYs, an 8051 MCU, and 192 KB of internal memory on a single die.

- **RoboSwitch selection guide (PDF):** [farnell.com/datasheets/2830681.pdf](https://www.farnell.com/datasheets/2830681.pdf)
- **RoboSwitch product brief:** [docs.broadcom.com/doc/5316X-PB100](https://docs.broadcom.com/doc/5316X-PB100)

### Broadcom BCM5315x ("Robo-2") — Next Generation

The successor chips replace the 8051 with an ARM Cortex-M7 and are designed to run an open-source RTOS. These may eventually appear in future Plus switch revisions.

- **BCM53154/53156/53158 datasheet (PDF):** [media.digikey.com — BCM53154,BCM53156,BCM53158.pdf](https://media.digikey.com/pdf/Data%20Sheets/Avago%20PDFs/BCM53154,BCM53156,BCM53158.pdf)

### Netgear Product Datasheets

- **GS110EMX / GS110MX datasheet (PDF):** [netgear.com — GS110EMX_GS110MX_DS.pdf](https://www.netgear.com/images/datasheet/switches/GS110EMX_GS110MX_DS.pdf)

---

## Security Research & Vulnerability Disclosures

### NCC Group — 15 Vulnerabilities in JGS516PE / GS116Ev2 (2021)

The most architecturally revealing security research on the Plus switch family. Manuel Ginés Rodríguez disclosed 15 CVEs exposing the unauthenticated debug endpoint, default-on TFTP server, firmware validation bypass, and NSDP authentication weaknesses. Netgear acknowledged that hardware limitations prevent implementing HTTPS or fixing fundamental protocol issues.

- **NCC Group technical advisory:** [Multiple Vulnerabilities in Netgear ProSAFE Plus JGS516PE / GS116Ev2 Switches](https://www.nccgroup.com/research-blog/technical-advisory-multiple-vulnerabilities-in-netgear-prosafe-plus-jgs516pe-gs116ev2-switches/)
- **Fox-IT mirror of the same advisory:** [fox-it.com — Technical Advisory](https://www.fox-it.com/be-en/technical-advisory-multiple-vulnerabilities-in-netgear-prosafe-plus-jgs516pe-gs116ev2-switches/)

### Key CVEs

| CVE | CVSS | Summary |
|---|---|---|
| CVE-2020-26919 | 9.8 | Unauthenticated RCE via hidden debug endpoint (`submitId=debug` in `login.htm`) |
| CVE-2020-35220 | 8.3 | TFTP server active by default with no authentication |
| CVE-2020-35232 | 8.1 | Firmware written to flash before validation; oversized images overwrite entire memory |
| CVE-2020-35231 | 8.8 | NSDP authentication bypass by skipping token request |

### Coverage & Analysis

- **PortSwigger / The Daily Swig:** [Critical RCE bug patched in Netgear ProSAFE Plus switches](https://portswigger.net/daily-swig/critical-rce-bug-patched-in-netgear-prosafe-plus-switches)
- **SecurityWeek:** [Unpatched Flaws in Netgear Business Switches Expose Organizations to Attacks](https://www.securityweek.com/unpatched-flaws-netgear-business-switches-expose-organizations-attacks/)
- **Security Affairs:** [Experts found 15 flaws in Netgear JGS516PE, including a critical RCE](https://securityaffairs.com/115586/hacking/netgear-soho-flaws.html)

### Netgear Security Advisory

- [Security Advisory for Multiple Vulnerabilities on Some ProSAFE Plus Switches](https://kb.netgear.com/000062993/Security-Advisory-for-Multiple-Vulnerabilities-on-Some-ProSAFE-Plus-Switches)

### Firmware Updates (examples of the small binary sizes)

- [GS108Ev4 Firmware Version 1.0.1.3](https://kb.netgear.com/000066315/GS108Ev4-Firmware-Version-1-0-1-3)
- [GS108Ev4 Firmware Version 1.0.1.9](https://kb.netgear.com/000068432/GS108Ev4-Firmware-Version-1-0-1-9)

---

## NSDP Protocol — Reverse Engineering & Tools

NSDP (Netgear Switch Discovery Protocol) is the proprietary Layer 2 management protocol used by Plus switches on UDP ports 63321–63324. It has been extensively reverse-engineered. The password "encryption" is XOR against the hardcoded key `"NtgrSmartSwitchRock"`.

### Protocol Documentation

- **Wikipedia:** [Netgear Switch Discovery Protocol](https://en.wikipedia.org/wiki/Netgear_Switch_Discovery_Protocol)
- **HandWiki:** [Netgear NSDP](https://handwiki.org/wiki/Netgear_NSDP)

### Open-Source NSDP Implementations

- **ProSafeLinux** (Python, ~180 commits): [openhub.net/p/prosafelinux](https://openhub.net/p/prosafelinux)
- **libnsdp** (C library): [github.com/AlbanBedel/libnsdp](https://github.com/AlbanBedel/libnsdp)
- **node-netgear-nsdp** (Node.js): [github.com/jue89/node-netgear-nsdp](https://github.com/jue89/node-netgear-nsdp)
- **go-nsdp** (Go library): [pkg.go.dev/github.com/hdecarne-github/go-nsdp](https://pkg.go.dev/github.com/hdecarne-github/go-nsdp)

### Network Discovery & Analysis Tools

- **nsdp-discover** (NCC Group Nmap NSE script): [github.com/nccgroup/nsdp-discover](https://github.com/nccgroup/nsdp-discover)
- **wireshark-nsdp** (Wireshark dissector plugin): [github.com/kamiraux/wireshark-nsdp](https://github.com/kamiraux/wireshark-nsdp)

### Netgear's Official Discovery Tool (NSDP successor)

- [NETGEAR Discovery Tool for Windows Version 2.0.4](https://kb.netgear.com/000066269/NETGEAR-Discovery-Tool-for-Windows-Version-2-0-4)

---

## Web Interface Scraping & Home Automation

### py-netgear-plus (Python library)

Provides programmatic access to the Plus switch web management interface via HTML scraping. Supports reading port statistics, PoE status, VLAN configuration, and more. Powers the Home Assistant integration below.

- **GitHub:** [github.com/foxey/py-netgear-plus](https://github.com/foxey/py-netgear-plus)

### Home Assistant Netgear Plus Integration

Brings Plus switch monitoring (port traffic, PoE power draw, link status) into Home Assistant dashboards. Tested with GS108PE v3, GS110EMX, GS305EP, GS308EP, and others.

- **GitHub:** [github.com/ckarrie/ha-netgear-plus](https://github.com/ckarrie/ha-netgear-plus)

---

## OpenWrt — What Works and What Doesn't

### Plus switches: No OpenWrt support (and likely never will be)

The Plus switches lack a general-purpose CPU, have insufficient RAM and flash, and run bare-metal 8051 firmware. OpenWrt cannot run on these devices. The OpenWrt forum thread on the GS308E v4 (linked above) documents this limitation.

### Pro switches: OpenWrt support exists

For reference, the higher-end Realtek-based Pro switches do run OpenWrt:

- **GS108T v3 OpenWrt support PR:** [github.com/openwrt/openwrt/pull/3709](https://github.com/openwrt/openwrt/pull/3709) — Adds support for the Realtek RTL8380M-based GS108T v3 to the `realtek` target. Merged into mainline.
- **OpenWrt Rpi4 community build thread** (discusses DSA switch management): [forum.openwrt.org/t/rpi4-community-build/69998/843](https://forum.openwrt.org/t/rpi4-community-build/69998/843)

---

## Product Pages & Specifications

- [NETGEAR GS110EMX on Amazon](https://www.amazon.com/NETGEAR-2x10-Gig-Multi-Gig-Lifetime-Protection/dp/B0765ZPY18)
- [NETGEAR GS108E on NetGuardStore](https://www.netguardstore.com/gs108e.asp)
- [NETGEAR GS305EP on NetGuardStore](https://www.netguardstore.com/gs305ep.asp)
- [NETGEAR GS110EMX on Netstore Direct](https://www.netstoredirect.com/other-netgear-switches/92504-netgear-gs110emx-0606449128932.html)

---

## Quick Reference: Architecture Summary

| Category | Plus / Smart Managed Plus | Smart Managed Pro |
|---|---|---|
| Examples | GS105E, GS108E, GS108PE, GS305EP, GS308EP, GS110EMX | GS108T, GS308T, GS110TP |
| CPU | 8051 MCU inside switch ASIC (or ARM on GS110EMX) | Dedicated MIPS or ARM SoC |
| RAM | 192 KB internal (or external DDR on GS110EMX) | 64–128 MB DDR |
| Flash | 64 KB – 4 MB SPI | 8–32 MB SPI/NAND |
| OS | Bare-metal / mask ROM RTOS | eCos or Linux |
| Web server | Custom proprietary | GoAhead or custom |
| CLI/SSH | None | Yes |
| SNMP | None | Yes |
| GPL sources | None published | Published |
| OpenWrt | Impossible | Supported (some models) |
| HTTPS | Cannot support (hardware limitation) | Supported |
