# Inside Netgear's cheap managed switches

**Netgear's low-end "Plus" and "Smart Managed Plus" switches run proprietary bare-metal firmware on a tiny 8051 microcontroller embedded inside the switch ASIC itself — not Linux, not eCos, and not any recognizable RTOS.** This makes them fundamentally different from the higher-end "Smart Managed Pro" models (GS108T, GS308T) that run eCos or Linux on dedicated MIPS/ARM CPUs with 64–128 MB of RAM. The Plus switches have as little as **64 KB of external SPI flash** and no general-purpose CPU at all, which means custom firmware like OpenWrt is structurally impossible on these devices. Netgear itself has acknowledged these hardware constraints, stating it **"cannot fix"** certain security vulnerabilities because the SoC lacks sufficient CPU power and memory for HTTPS.

---

## Four hardware generations, one common pattern

The Plus switch lineup spans roughly four generations of silicon, but they all share one architectural trait: the management logic runs on a microcontroller integrated directly into the Ethernet switch chip. No separate CPU is needed.

**Generation 1 (pre-2012)** used a two-chip design: a **Broadcom BCM53114 or BCM53118** switch ASIC paired with a **Taifatech TF-480-ACL** external management microcontroller. The TF-480 is a Taiwanese fabless chip purpose-built for switch management UIs. This architecture appeared in the GS105E v1 and GS108PE v1.

**Generation 2 (2012–2018)** consolidated to a single chip: the **Broadcom BCM53128**, which integrates an 8-port gigabit switch with EEE PHYs and an **Intel 8051 microcontroller core** plus **192 KB of internal memory**. The 8051 reads its firmware from a tiny external SPI flash (as small as 64 KB), calls into a **mask-programmed internal ROM** containing an RTOS and CLI, and handles the entire web management stack autonomously. This chip powers the GS108E v3, GS108PE v3, GS116E v2, and likely the GS105PE. The PoE variants add a **Broadcom BCM59101** quad 802.3af PSE controller as a companion chip.

**Generation 3 (2017+, multi-gigabit)** is an outlier. The **GS110EMX** requires three chips to deliver 10GBASE-T: a **Marvell 88F6811** ARM Cortex-A9 CPU (Armada 38x family), a **Marvell 88E6390X** Link Street switch ASIC, and a **Marvell 88X3310P** 10G PHY. It has external Nanya DDR3 RAM and two **Macronix MX25V2006E** SPI flash chips (256 KB each). This is architecturally closer to the Pro switches but still ships with only a web interface.

**Generation 4 (2019+)** moved to Realtek silicon. The GS308E v4 uses a **Realtek RTL8370N** with an integrated 8051 MCU and a larger **Winbond W25Q32** SPI flash (4 MB). The GSS108E "Click Switch" uses an **RTL8370**. The newer GS305EP and GS308EP PoE models have not been publicly torn down but likely use this same Realtek family. Looking ahead, Broadcom's next-generation "Robo-2" chips (BCM5315x) replace the 8051 with an **ARM Cortex-M7** and run an open-source RTOS.

| Model | Switch chip | Management CPU | Flash | PoE IC |
|---|---|---|---|---|
| GS105E v1 | Broadcom BCM53114 | Taifatech TF-480 | SPI EEPROM | — |
| GS108PE v1 | Broadcom BCM53118 | Taifatech TF-480 | SPI flash | External |
| GS108PE v3 | Broadcom BCM53128 | Integrated 8051 | ~64 KB SPI | BCM59101 |
| GS108E v3 | Broadcom (BCM531xx) | Integrated 8051 | 2 MB (W25Q16) | — |
| GS116E v2 | Broadcom (BCM531xx) | Integrated 8051 | SPI flash | — |
| GS308E v4 | Realtek RTL8370N | Integrated 8051 | 4 MB (W25Q32) | — |
| GSS108E | Realtek RTL8370 | Integrated 8051 | SPI flash | — |
| GS110EMX | Marvell 88E6390X | Marvell 88F6811 (ARM) | 2× 256 KB SPI | — |
| GS305EP/GS308EP | Unknown (likely Realtek) | Integrated MCU | SPI flash | Dedicated PSE IC |

---

## Firmware is bare-metal, not Linux or eCos

The most detailed public reverse engineering of this firmware class comes from **florolf's 2019 analysis** of the BCM53128 inside a Netgear GS108 v4. Dumping the 64 KB SPI flash revealed standard 8051 machine code starting with an `LJMP 0x00EF` instruction at the reset vector. Strings extracted from the binary include `"53128 Gigabit PHY Driver"`, `"5464 Gigabit PHY Driver"`, version stamps (`"Oct 27 2011"`), and assertion macros — evidence of a structured codebase, but no identifiable third-party RTOS signatures.

The 8051 firmware does not stand alone. It makes **extensive calls into a mask ROM** burned into the BCM53128 during fabrication. Florolf extracted this ROM by writing custom 8051 code that bit-banged the contents out through GPIO pins. The internal ROM contains a **complete RTOS with task scheduling and a hidden CLI** — a development/debug environment that Broadcom never documents publicly. The 8051 accesses switch registers directly, managing VLAN tables, QoS queues, IGMP snooping, port mirroring, and the web interface from this single core.

**Firmware update files are small `.bin` archives** (typically 1–3 MB) distributed in `.zip` containers — orders of magnitude smaller than the 8–32 MB images on Linux-based Pro switches. No GPL source code is published for any Plus switch, which is strong negative evidence against Linux (Netgear publishes GPL sources for all its Linux-based products). The web server is **custom and proprietary**, with no evidence of GoAhead, Boa, lighttpd, or any other known embedded HTTP server in firmware string dumps. For the GS308E v4, an OpenWrt forum user who dumped the RTL8370N firmware in January 2025 described it as "quite compressed and obfuscated" with essentially no public documentation for the chipset.

---

## NSDP: the management protocol Netgear wishes would go away

Beyond the web interface, these switches speak **NSDP (Netgear Switch Discovery Protocol)** — a proprietary Layer 2 protocol on UDP ports 63321–63324 using TLV-encoded messages. The protocol has been **thoroughly reverse-engineered** by the community, most notably because its password "encryption" is a trivial **XOR against the hardcoded string `"NtgrSmartSwitchRock"`**. Multiple open-source implementations exist: **ProSafeLinux** (Python, ~180 commits), **libnsdp** (C), **node-netgear-nsdp** (Node.js), two independent Go libraries, an Nmap NSE discovery script from NCC Group, and a Wireshark dissector plugin.

NSDP also enables firmware updates by triggering a TFTP server on the switch. Netgear has declared NSDP end-of-life and disabled it by default in newer firmware, replacing it with UPnP-based discovery. The **py-netgear-plus** library on GitHub provides Python access to the web management interface via HTML scraping, powering a Home Assistant integration that supports the GS108PE v3, GS110EMX, GS305EP, GS308EP, and many others.

---

## 15 CVEs exposed the architecture's limits

The most revealing security research came from **NCC Group's Manuel Ginés Rodríguez**, who disclosed **15 vulnerabilities** in the JGS516PE and GS116E v2 between 2020 and 2021. These findings exposed deep architectural details:

- **CVE-2020-26919** (CVSS **9.8**, Critical): The web application contained a hidden debug endpoint that accepted system commands via POST requests — exploitable without any authentication by sending `submitId=debug` to `login.htm`.
- **CVE-2020-35220** (CVSS 8.3): The **TFTP server was active by default** with no authentication, allowing anyone on the LAN to upload arbitrary firmware with a single `atftp put` command.
- **CVE-2020-35232** (CVSS 8.1): Firmware validation was bypassable because the firmware length was read from the image header (not the actual file), and images were **written to flash before validation**. The maximum accepted size (0x17FF00 ≈ 1.5 MB) exceeded the actual partition size (0xC0000 = 768 KB), allowing an attacker to overwrite the entire memory space with custom code.
- **CVE-2020-35231** (CVSS 8.8): NSDP authentication could be bypassed by skipping the random-token request step, causing the switch to accept an empty authentication hash.

Netgear's response was telling: they stated they **cannot implement HTTPS** or fix the fundamental NSDP weaknesses "due to hardware limitations" — the 8051 MCU and its tiny memory footprint simply cannot support TLS. Only the most critical bugs (RCE, unauthenticated firmware upload) were patched in firmware v2.6.0.48. These vulnerabilities affect the entire Plus switch family architecturally, not just the specific models tested.

---

## Reverse engineering is sparse but growing

Community RE efforts on the Plus switches specifically remain limited, largely because the 8051-based architecture offers little payoff — there is no Linux-capable CPU to run custom firmware on. The most significant work includes:

- **florolf's BCM53128 analysis** (blog.n621.de, 2019) produced a custom 8051 emulator, an internal ROM dump, and the **neatgear** tool on GitHub that patches SPI flash dumps to add 802.1Q VLAN configuration to the unmanaged GS108.
- **alessandromrc's GS308E v4 teardown** (OpenWrt forum, January 2025) identified the RTL8370N chip, found a UART interface at **56000 baud 8N1**, and dumped the firmware to GitHub. Boot menu output was visible on UART but provided no useful shell.
- **The "Switcher" project** on Hackaday.io attaches a USB interface module to unmanaged Netgear GS108 switches and leverages the Linux kernel's DSA (Distributed Switch Architecture) framework with its existing B53 and RTL8365MB drivers to manage them from a host computer.

For context, the higher-tier **GS108T v2** (Smart Managed Pro) has far richer RE documentation: fvollmer's GitHub project documents full UART access (9600 baud under the glued-on heatsink), CFE bootloader interruption, eCos source builds, and a partial OpenWrt port that boots but lacks working networking due to SSB bus driver issues. The **GS108T v3** and **GS110TPP** (Realtek RTL8380M, Linux-based) have full OpenWrt support in mainline since 21.02.

A cross-platform discovery in January 2026 demonstrated how generic the RTL8370N platform is: a user wrote the GS308E firmware to a TP-Link TL-SG108 v9.6 (also RTL8370N-based, sold as unmanaged) and it **booted with Netgear's management interface**, proving these devices are essentially identical silicon with different brand labels.

---

## No FCC photos exist for wired switches

One common research dead end: **these switches have no FCC ID filings** because wired-only Ethernet devices contain no intentional radio transmitters. They comply with FCC Part 15 Subpart B (Class B unintentional emissions) via a Declaration of Conformity, which does not require an FCC ID or the submission of internal photos. Netgear's FCC grantee code "PY3" applies exclusively to wireless products.

---

## Conclusion

The cheap Netgear Plus switches are **remarkably minimal devices**: a single switch ASIC with an embedded 8051 microcontroller, tens of kilobytes of SPI flash, and proprietary firmware that calls into a mask-programmed ROM for its RTOS primitives. This architecture delivers VLANs, QoS, PoE management, and a web GUI from hardware that costs under $30, but it comes with fundamental security constraints that Netgear has acknowledged are unfixable. The newer Broadcom Robo-2 chips (ARM Cortex-M7) may eventually bring enough headroom for TLS and more robust management stacks, but for now, the Plus line occupies an unusual niche — too complex to ignore security-wise, too simple to remediate. Anyone considering deep RE of these devices should start with florolf's BCM53128 work and the NCC Group advisory as the two richest public sources, and temper expectations about custom firmware: without a general-purpose CPU, there is no Linux to load.
