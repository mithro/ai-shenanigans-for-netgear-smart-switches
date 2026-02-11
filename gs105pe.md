# Accessing the MAC Address Table on the Netgear GS105PE

## The Problem

The Netgear GS105PE is a 5-port Gigabit Plus (Smart Managed Plus) switch with PoE pass-through. Like all Plus switches, it maintains a full MAC address forwarding table (ARL) in hardware — the switch ASIC learns source MACs and associates them with ingress ports to perform L2 forwarding at line rate. However, Netgear does not expose this table through any official interface. A Netgear community moderator confirmed in 2017 (still true as of 2025) that Plus switches lack an "Address Table" feature in the web UI. The recommended path for MAC table visibility is upgrading to a GS108T or higher-tier Smart Managed Pro switch.

This document catalogues every known approach to reading the ARL table on the GS105PE.

---

## Hardware Background

The GS105PE is a Generation 2 Broadcom-based Plus switch. It shares a firmware codebase and user manual with the GS105E v2, GS108E v3, GS108PE v3, and GS116E v2. No published teardown identifies the exact chip, but based on the product generation and Broadcom's 5-port RoboSwitch lineup, it almost certainly contains a **BCM53125** (or possibly BCM53115). The 8-port equivalents in this generation (GS108E v3, GS108PE v3) use the BCM53128.

The architecture is bare-metal firmware running on an **8051 MCU embedded within the Broadcom switch ASIC**. There is no Linux, no traditional RTOS — just proprietary 8051 code loaded from a small SPI flash (64KB in the GS108 v4; likely similar in the GS105PE). The 8051 handles the web interface, NSDP protocol, and configuration, while the ASIC hardware performs all L2 forwarding independently.

The ARL table is a 4-bin × 1,024-bucket structure (4,096 entries), maintained entirely in the ASIC's internal SRAM. The silicon learns MACs, ages them, and forwards frames without any involvement from the 8051 firmware. The firmware simply never reads or exposes the table.

---

## Approach 1: Official Interfaces

### Web Interface

The web UI exposes port status, port statistics (byte/packet counters), VLANs, QoS, IGMP snooping, PoE settings, loop detection, and port mirroring. No page or hidden endpoint exists for viewing learned MAC addresses. The `py-netgear-plus` library (used for Home Assistant integration) confirms there is nothing to scrape.

### NSDP Protocol

The Netgear Switch Discovery Protocol uses TLV-encoded messages over UDP. Known TLV types cover device identity, port status, port statistics, VLAN configuration, and similar. No TLV type for MAC/FDB table queries has been documented across any NSDP reimplementation (ProSafeLinux, libnsdp, go-nsdp). NCC Group's security research fuzzed the protocol and found undocumented TLVs but nothing for ARL access.

**Verdict: Not possible. The feature does not exist in any official interface.**

---

## Approach 2: Exploiting Known Firmware Vulnerabilities

NCC Group disclosed 15 CVEs affecting Plus switches in 2020–2021. All were confirmed on the JGS516PE and GS116Ev2, which share the same Generation 2 Broadcom platform and firmware codebase as the GS105PE. Applicability to the GS105PE is unverified but likely.

### Debug Command Execution (CVE-2020-26919, CVSS 9.8)

On firmware versions prior to 2.6.0.43, the web application failed to implement access controls on a debug endpoint. Any page accepts POST requests with `submitId=debug` and a `debugCmd` parameter:

```bash
curl -X POST --data-raw 'submitId=debug&debugCmd=sys+dump&submitEnd=' 'http://<switch-IP>/login.htm'
```

This invokes pre-existing debug handlers compiled into the firmware. The only documented command is `sys dump` (memory dump). This is not arbitrary code execution — `debugCmd` dispatches to a fixed set of diagnostic routines baked into the firmware. Nobody has published the full list of available commands, and no known command reads the ARL table.

However, this is the lowest-effort avenue worth investigating: if Netgear's engineers included an ARL dump command (plausible, given it is a debug interface for a switch), it could provide remote ARL access with a single HTTP request on pre-patch firmware.

**Status**: Patched in firmware v2.6.0.43. Unverified on GS105PE. Debug command set undocumented.

### Unauthenticated Firmware Upload (CVE-2020-35220, CVSS 8.3) + Validation Bypass (CVE-2020-35232, CVSS 8.1)

The TFTP-based firmware update mechanism allows uploading custom firmware without authentication (CVE-2020-35220) and does not properly validate firmware length or checksums (CVE-2020-35232). The uploaded file is written directly to the SPI flash before validation, allowing an attacker to overwrite the entire flash with custom code.

In theory, one could craft a firmware image containing custom 8051 code that reads the ARL Page 05h registers and outputs the results. In practice, nobody has done this because:

- Writing valid 8051 firmware requires understanding the boot sequence, memory map, XIP caching, bank switching, and the register interface between the 8051 and the switch ASIC
- Florolf (the only person who has deeply reverse-engineered this 8051 firmware) built an emulator and still couldn't fully figure out the firmware's interaction with switch registers before giving up
- Flashing bad firmware bricks the switch with no recovery path
- Both CVEs were patched in firmware v2.6.0.48

### Buffer Overflows (CVE-2020-35225, CVE-2020-35227)

Authenticated buffer overflow vulnerabilities in the web interface that enable code execution. These could theoretically inject shellcode that reads ARL registers, but the 8051's tiny memory model (128 bytes internal RAM, banked XRAM) makes traditional shellcode impractical.

**Verdict: No exploit has been publicly demonstrated that reads the ARL table. The debug command set (CVE-2020-26919) is the most promising low-effort angle — worth probing on pre-patch firmware. The firmware upload path requires 8051 reverse engineering that remains incomplete.**

---

## Approach 3: UART Access to Broadcom Internal ROM CLI

Florolf's reverse engineering of the BCM53128 (in a Netgear GS108 v4) uncovered a significant finding: the Broadcom ASIC's internal ROM — separate from Netgear's firmware in SPI flash — contains an RTOS and a complete diagnostic CLI. This is Broadcom's own debug shell baked into the silicon, present on every BCM53128 and (almost certainly) every BCM53125.

### What Was Discovered

Florolf dumped the SPI flash, reverse-engineered Netgear's 8051 firmware, then wrote custom 8051 code to bit-bang the internal ROM contents out through GPIO pins. The ROM revealed:

- A Broadcom RTOS
- A complete CLI for interacting with the ASIC
- UART access via standard 8051 UART SFRs
- TX and RX on **chip pins 69 and 70** (confirmed via a Broadcom reference design where these pins are broken out to a header labelled "for debug only")
- Baud rate: 9600 8N1 (changes to approximately 16000 baud when no Ethernet link is detected)

### Why This Matters

Broadcom's internal debug CLIs on managed switch ASICs typically include register read/write commands and ARL table dump functionality. If the BCM53125/53128 ROM CLI follows this pattern, commands for ARL iteration or register reads at Page 05h would provide direct MAC table access with no custom code required — just a UART connection.

### Procedure

1. Open the GS105PE case
2. Locate chip pins 69 (TX) and 70 (RX) on the BCM53125 QFP package, or find corresponding PCB test points/vias
3. Solder fine wires to these pins
4. Connect to a 3.3V USB-UART adapter
5. Open a serial terminal at 9600 8N1
6. Explore the CLI command set (try `help`, `?`, or common Broadcom debug commands like `arl dump`, `reg read`, etc.)

### Limitations

- Physically invasive — requires soldering to QFP pins on the ASIC
- The CLI command set has not been published; florolf did not document it
- The CLI may require authentication or may not include ARL-specific commands
- Incorrect voltage levels could damage the chip
- The GS105PE's PCB layout may not have convenient test points for these pins

### State of Florolf's Research

Florolf's work remains the deepest public exploration of the BCM53128 8051 side. He built an emulator, got the firmware showing signs of life, confirmed the firmware touches switch registers, but ultimately gave up due to the complexity of 8051 bank switching. He explicitly noted that the 8051 core "can also access the switch's datapath, which might allow one to implement features such as LLDP or even proper STP" — confirming the 8051 has the register access needed for ARL reads, even if nobody has built firmware to do it.

**Verdict: Most promising unexplored software-based path. If the Broadcom ROM CLI includes ARL commands, this yields MAC table access with only a UART connection and no custom firmware. The command set is unknown and requires physical investigation.**

---

## Approach 4: SPI Bus + Linux b53 DSA Driver

This is the **only method that has been fully demonstrated** for reading the ARL table on a BCM53128-based switch.

### How It Works

The BCM53125/53128 exposes its full register set via an SPI slave interface. By connecting an external SPI master to the chip's SPI bus, a Linux host uses the mainline kernel's **b53** DSA (Distributed Switch Architecture) driver to manage the switch as a directly-attached network ASIC. The b53 driver explicitly supports both BCM53125 and BCM53128 with identical parameters (`arl_bins=4`, `arl_buckets=1024`).

### ARL Table Registers

The entire Broadcom RoboSwitch family shares the same ARL access registers at Page 05h:

| Register | Address | Function |
|---|---|---|
| ARL Read/Write Control | Page 05h, Addr 00h | Triggers read/write/search operations |
| MAC Address Index | Page 05h, Addr 02h | MAC address to look up |
| VLAN ID Index | Page 05h, Addr 08h | VLAN for lookup |
| ARL Table Data Entry N | Page 05h, Addr 18h+ | Port mapping, VLAN, static/dynamic, age |
| ARL Search Control | Page 05h, Addr 50h | Initiate iteration of all entries |
| ARL Search Address | Page 05h, Addr 51h | Next MAC during search |

This register layout is identical across BCM53115, BCM53125, and BCM53128. The b53 driver handles minor address variations automatically.

### Demonstrated Implementation (Switcher Project)

The Switcher project (github.com/hvegh/Switcher) demonstrated this on a Netgear GS108 v3 with a BCM53128.

**Hardware**:
- CH341A USB-to-SPI adapter (~€4 / ~AU$7)
- 4 wires (SPI CLK, MOSI, MISO, CS)
- Soldering iron

**Procedure**:

1. Open the switch case
2. Locate SPI CLK, MOSI, MISO, CS pins on the BCM53125 package (or find PCB test points — the SPI flash is also on this bus and may have more accessible pads)
3. Solder 4 wires to the SPI bus
4. Connect to CH341A USB-SPI adapter on a Linux host
5. Load kernel modules:
   ```bash
   modprobe spi_ch341_usb
   modprobe b53_common
   modprobe b53_spi
   ```
6. The b53 driver auto-detects the chip ID during probe and binds
7. Read the ARL table:
   ```bash
   bridge fdb show
   ```

The output provides MAC address, VLAN ID, port number, and static/dynamic status for every learned entry.

An OpenWrt swconfig patch (by Alexandru Ardelean, 2015) also provides ARL access:
```bash
swconfig dev switch0 set arl "rd XX:XX:XX:XX:XX:XX vid NNNN"
```
This returns the MAC, VLAN ID, valid/age/static flags, and port number for a specific entry.

### GS105PE-Specific Considerations

The only work specific to the GS105PE is locating the SPI pins on its PCB. The board layout differs from the GS108 v3 that the Switcher project documents. Options:

- **Chip pins directly**: Identify SPI CLK/MOSI/MISO/CS on the QFP package from the BCM53125 datasheet pinout
- **SPI flash pads**: The SPI flash chip shares the same SPI bus and is typically an 8-pin SOIC package with larger, more accessible pads. Tapping the bus at the flash chip may be easier than soldering to the ASIC
- **PCB test points**: Some boards have unpopulated pads or vias on the SPI traces

After opening the case and confirming the chip marking (BCM53125 vs BCM53115), cross-reference the datasheet pinout with the PCB traces.

### Limitations

- Physically invasive — requires opening the case and soldering
- Requires a dedicated Linux host connected via USB
- The SPI connection is local/physical, not remote
- Voids warranty
- The Linux host must remain connected to read the table
- Potential bus contention with the on-board 8051 (which also accesses registers via the internal bus) — in practice this has not been reported as a problem with the Switcher project

**Verdict: Proven working. The only fully demonstrated method. Uses well-supported Linux kernel infrastructure. The main effort is physical — identifying SPI pin locations on the GS105PE PCB and soldering 4 wires.**

---

## Approach 5: Passive Network Inference

These methods require no physical modification but provide only indirect MAC-to-port mapping.

### Port Mirroring + Packet Capture

The GS105PE supports port mirroring via its web interface. Mirror each port (one at a time) to a monitoring port, then capture traffic and extract source MACs.

```bash
# After configuring port X mirrored to monitoring port via web UI:
tcpdump -i eth0 -e -c 1000 | awk '{print $2}' | sort -u
```

Repeat for each port to build a MAC-to-port map.

**Pros**: No hardware modification, works with existing features.
**Cons**: Must mirror each port individually (only one source port can be mirrored at a time on most Plus switches). Only captures MACs from hosts actively transmitting. Misses idle devices.

### ARP Scan + Port Statistics Correlation

The web interface exposes per-port byte and packet counters. By generating targeted traffic and watching which port's counters increment, you can correlate MACs to ports.

1. Record per-port packet counters (via web UI or NSDP query)
2. Send a unicast frame to a specific MAC (e.g., an ARP request to a known IP)
3. Re-read per-port counters
4. The port whose TX counter incremented is where that MAC is connected

This can be scripted using `py-netgear-plus` or NSDP tools for counter reads and `arping` for traffic generation.

**Pros**: Non-invasive, scriptable, no hardware changes.
**Cons**: Slow (one MAC at a time). Unreliable if other traffic is flowing simultaneously. Requires knowing target IPs/MACs in advance. Cannot discover unknown devices.

### LLDP Snooping

If connected devices transmit LLDP (Link Layer Discovery Protocol) frames, these can be observed from any port. LLDP frames include the sender's MAC address, port ID, system name, and other identification.

**Pros**: Rich device identification for LLDP-capable devices.
**Cons**: Only works if devices support LLDP. Consumer devices generally don't. Tells you neighbor identity, not the switch's internal FDB state.

**Verdict: Functional but indirect and incomplete. Port mirroring is the most reliable passive method. Best used when hardware modification is not an option.**

---

## Summary

| Approach | Works? | Effort | Requirements |
|---|---|---|---|
| Web UI / NSDP | ❌ No | — | — |
| Debug commands (CVE-2020-26919) | ❓ Unknown | Low | Pre-patch firmware, HTTP access |
| Firmware upload (CVE-2020-35220/35232) | ❓ Theoretically | Very High | 8051 RE, pre-patch firmware |
| UART to Broadcom ROM CLI | ❓ Promising | Medium | Solder to chip UART pins |
| SPI + b53 Linux driver | ✅ **Proven** | Medium | CH341A adapter, solder 4 wires, Linux host |
| Port mirroring + capture | ⚠️ Indirect | Low | tcpdump, patience |
| ARP scan + counter correlation | ⚠️ Indirect | Low | Scripting, known target MACs |

The **SPI + b53 driver** approach is the only method with a confirmed working demonstration. The **Broadcom ROM CLI via UART** is the most promising unexplored path — it could potentially provide ARL access with less physical modification than full SPI wiring if the CLI includes the right commands. The **debug command endpoint** is worth probing on pre-patch firmware as the absolute lowest-effort option, though its command set is undocumented and it may not include ARL functionality.
