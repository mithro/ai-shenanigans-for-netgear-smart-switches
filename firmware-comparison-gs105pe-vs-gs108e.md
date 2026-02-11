# Firmware Comparison: GS105PE vs GS108E Architecture Analysis

## Executive Summary

Binary analysis of all published GS105PE firmware (10 versions) against GS108Ev2 and GS108Ev3
firmware reveals that **all three product lines share the same 8051 MCU architecture** compiled with
the **Keil C51 toolchain**, but they run on **different switch silicon**:

| Model | Switch Silicon | MCU | Firmware Header |
|-------|---------------|-----|-----------------|
| **GS105PE** | **Realtek RTL8367N/RTL8367B** | 8051 | `0x12345678` magic (20-byte header) |
| **GS108Ev2** | **Broadcom BCM53128 + BCM5464** | 8051 | `UMHD` magic (embedded in IVT) |
| **GS108Ev3** | **Broadcom BCM53128 + BCM5464** | 8051 | `UMHD` magic (embedded in IVT) |

**Key finding: The GS105PE does NOT use Broadcom silicon.** Despite being in the same Netgear
"Plus" switch product family as the GS108E, the GS105PE uses Realtek RTL8367N/B switch SoCs.
However, both chip families embed an 8051 MCU core, so the firmware development approach,
toolchain, and much of the application-layer code is shared.

---

## 1. Definitive 8051 Architecture Evidence (All Three Models)

### 1.1 Reset Vector and Interrupt Vector Table

All three firmwares contain the classic 8051 Interrupt Vector Table (IVT) with LJMP (opcode `0x02`)
instructions at the standard addresses:

| IVT Entry | Address | GS108Ev2 | GS108Ev3 | GS105PE (offset +0x14) |
|-----------|---------|----------|----------|------------------------|
| RESET | 0x0000 | `02 3D 54` → LJMP 0x3D54 | `02 00 5E` → LJMP 0x005E | `02 10 16` → LJMP 0x1016 |
| INT0 | 0x0003 | `02 3C 90` → LJMP 0x3C90 | `02 4C D2` → LJMP 0x4CD2 | `02 10 09` → LJMP 0x1009 |
| Timer0 | 0x000B | `02 39 6E` → LJMP 0x396E | `02 49 29` → LJMP 0x4929 | `02 2E CF` → LJMP 0x2ECF |
| INT1 | 0x0013 | — | — | `02 31 14` → LJMP 0x3114 |

Note: GS105PE code starts at file offset 0x14 due to its 20-byte firmware header. The 8051 code
addresses shown are relative to the IVT start.

### 1.2 Keil C51 IRAM Initialization

All three firmwares contain the unmistakable Keil C51 startup sequence (`STARTUP.A51`):

```assembly
; GS108Ev2 at 0x3D54 / GS108Ev3 at 0x005E / GS105PE at 0x0016
MOV   R0, #0x7F       ; 78 7F / A8 7F  -- point to top of 128-byte IRAM
CLR   A                ; E4             -- clear accumulator
MOV   @R0, A           ; F6             -- clear IRAM byte
DJNZ  R0, $-1          ; D8 FD          -- loop until R0 = 0
MOV   SP, #xx          ; 75 81 xx       -- set stack pointer
```

The stack pointer differs between implementations:

| Model | Stack Pointer | Significance |
|-------|--------------|--------------|
| GS108Ev2 | `SP = 0x32` | Leaves more IRAM for direct-addressable variables |
| GS108Ev3 | `SP = 0x32` | Same as GS108Ev2 |
| GS105PE | `SP = 0x7F` | Uses top of IRAM for stack (Realtek's 8051 variant has more XDATA) |

### 1.3 Non-Standard SFR Writes (Chip-Specific 8051 Extensions)

After the IRAM clear, each firmware writes to chip-specific Special Function Registers:

**GS108Ev2/v3 (Broadcom 8051):**
```assembly
MOV   0x22, #0xA0/#0xB0    ; Broadcom-specific SFR
MOV   0x23, #0x00           ; Broadcom-specific SFR
```

**GS105PE (Realtek 8051):**
```assembly
MOV   0xB9, #0x00           ; Realtek-specific SFR
MOV   0xBA, #0x80           ; Realtek-specific SFR
MOV   0x96, #0x03           ; Watchdog control register
```

The SFR addresses differ because Broadcom and Realtek implement different peripheral mappings
on their 8051 cores.

### 1.4 8051 Opcode Frequency Distribution (First 64KB)

| Opcode | Instruction | GS108Ev2 | GS108Ev3 | GS105PE |
|--------|------------|----------|----------|---------|
| 0x02 | LJMP | 1,145 | 1,382 | 1,229 |
| 0x12 | LCALL | 3,379 | 3,407 | 2,783 |
| 0x22 | RET | 985 | 419 | 684 |
| 0x90 | MOV DPTR,#imm16 | — | — | 3,053 |
| 0xE0 | MOVX A,@DPTR | — | — | 1,560 |
| 0xF0 | MOVX @DPTR,A | — | — | 1,781 |
| 0x75 | MOV direct,#imm | 254 | 323 | 705 |

The high frequency of MOVX instructions (external memory access) is characteristic of 8051 code
that performs heavy I/O with memory-mapped switch registers.

---

## 2. Switch Silicon Identification

### 2.1 GS105PE: Realtek RTL8367N/RTL8367B

The GS105PE firmware contains **explicit Realtek chip model strings**:

```
RTL8367N                                    # In chip configuration metadata
RTL8367B                                    # In debug/diagnostic menu
RTK_MAX_NUM_OF_PRIORITY                     # Realtek SDK macro
RTK_MAX_NUM_OF_QUEUE                        # Realtek SDK macro
Macro RTK_MAX_NUM_OF_PRIORITY != SAL_MAX_NUM_OF_PRIORITY   # Debug assertion
```

The RTL8367N/B is a Realtek 5+2 port Gigabit Ethernet switch controller with an **embedded 8051
MCU core**. It integrates:
- 8051-compatible CPU
- 5-port Gigabit PHY
- Ethernet switch fabric
- SPI flash interface
- Memory-mapped switch registers accessible via MOVX (XDATA)

The "B" and "N" designations likely represent different silicon revisions used across GS105PE
hardware versions.

### 2.2 GS108Ev2/v3: Broadcom BCM53128 + BCM5464

Both GS108E firmware binaries contain explicit Broadcom chip references:

```
53128 Gigabit PHY Driver                    # Broadcom BCM53128 (main switch SoC)
5464 Gigabit PHY Driver                     # Broadcom BCM5464 (external quad PHY)
```

The BCM53128 is a Broadcom RoboSwitch-family 8-port managed gigabit switch SoC with an
**embedded 8051 MCU core**. The BCM5464 provides external PHY transceivers for 4 of the 8 ports
(the BCM53128 has 4 integrated PHYs + 4 SGMII interfaces).

### 2.3 Why Different Silicon?

The GS105PE (5-port with PoE pass-through) and GS108E (8-port) have different port counts and
feature sets, making different switch SoCs economically sensible:

| Requirement | GS105PE | GS108E |
|------------|---------|--------|
| Port count | 5 GbE | 8 GbE |
| PoE | Pass-through on port 1 | None (v2) / PoE variant (GS108PEv2) |
| Chip solution | Single Realtek RTL8367N | BCM53128 + external BCM5464 |
| Cost structure | Single-chip (lower BOM) | Two-chip (higher BOM, more ports) |

---

## 3. Firmware Format Differences

### 3.1 GS105PE: 20-Byte Magic Header

```
Offset  Size  Field               Example (V1.6.0.17)
------  ----  -----               -------------------
0x00    4     Magic               0x12345678
0x04    4     Payload size        0x000A51AC (676,268 = filesize - 20)
0x08    4     Unknown             0x0000055D (1,373)
0x0C    4     Checksum            0x03D2A222
0x10    4     Trailer/padding     0x332255FF
0x14    ...   8051 code begins    02 10 16 (LJMP 0x1016)
```

- Magic `0x12345678` is consistent across all 10 firmware versions
- Payload size field always equals `file_size - 20`
- Bootloader validates with: `Hdr Chksum Error` / `PayLoad Chksum Error`
- Linear code image (no 64KB banking visible in the file format)

### 3.2 GS108Ev2/v3: UMHD Inline Header

```
Offset  Size  Field               GS108Ev2            GS108Ev3
------  ----  -----               --------            --------
0x00    3     LJMP (reset)        02 3D 54            02 00 5E
0x03    3     LJMP (INT0)         02 3C 90            02 4C D2
0x06    3     AJMP                01 00 5E            01 00 5E
0x0E    4     Magic               "UMHD"              "UMHD"
0x12    16    Hex metadata        "0004000000008068"  "000B000000009C6B"
0x28    10    Model name          "GS108Ev2\0\0"      "GS108Ev3\0\0"
0x42    3     Version (BCD)       01 00 0C            02 06 18
```

- The UMHD header is **embedded within the 8051 IVT** (between interrupt vectors)
- The firmware is a direct memory image (no wrapper)
- 64KB code banking: v2 uses 4 banks (256 KB), v3 uses 11 banks (704 KB)

### 3.3 Code Organization

| Property | GS105PE | GS108Ev2 | GS108Ev3 |
|----------|---------|----------|----------|
| Image layout | Linear + 20-byte header | 4 x 64KB banked | 11 x 64KB banked |
| Total code size | 660-858 KB | 256 KB (exact) | 704 KB |
| Web UI | Yes (full HTML/CSS/JS) | No (NSDP only) | Yes (full HTML/CSS/JS) |
| TCP/IP stack | LWPS (custom) | uIP | uIP |
| String density | 45.3% printable | 21.4% printable | 45.0% printable |

---

## 4. Shared Software Architecture

Despite different switch silicon, the firmware shares significant common code:

### 4.1 Source Tree Structure (from GS105PE debug strings)

```
common/
└── src/
    ├── app/
    │   ├── web/
    │   │   ├── httpd.c          # Embedded HTTP server
    │   │   └── web_api.c        # CGI handler API
    │   ├── dhcpc/
    │   │   └── dhcpc.c          # DHCP client
    │   ├── scc/
    │   │   ├── scc.c            # Switch Configuration Controller (NSDP)
    │   │   └── scc_tftp.c       # TFTP-based firmware update via SCC
    │   ├── tftp/
    │   │   └── tftpc.c          # TFTP client
    │   └── lldp/
    │       └── lldp.c           # Link Layer Discovery Protocol
    └── lwps/
        ├── etharp.c             # ARP handler
        ├── icmp.c               # ICMP/ping handler
        ├── igmp_snooping.c      # IGMP snooping
        ├── lwps_api.c           # Network stack API
        ├── tcp.c                # TCP implementation
        └── udp.c                # UDP implementation
```

### 4.2 SAL (Switch Abstraction Layer)

The GS105PE firmware exposes a `sal_*` function namespace that abstracts the switch chip:

```
sal_vlan_config_restore()
sal_port_config_restore()
sal_qos_config_restore()
sal_rate_config_restore()
sal_mirror_config_restore()
sal_trunk_config_restore()
sal_igmp_restore()
sal_loop_config_restore()
sal_sys_config_restore()
sal_eee_restore()
sal_green_restore()
sal_ipauth_config_restore()
```

This SAL layer is what allows the same application code to run on both Realtek and Broadcom
switch SoCs — the SAL provides a unified API while the chip-specific driver talks to the
hardware registers.

### 4.3 Common Protocol Support

All three firmwares support:
- **NSDP** (Netgear Switch Discovery Protocol) for ProSAFE Plus Utility management
- **DHCP client** for IP address acquisition
- **LLDP** for neighbor discovery
- **IGMP snooping** for multicast management
- **TFTP** for firmware updates
- **HTTP** (GS105PE and GS108Ev3 only) for web-based management

### 4.4 Common Features (from CGI endpoints in GS105PE)

```
/login.cgi, /logout.cgi                    # Authentication
/switch_info.cgi                            # System information
/port_setting.cgi, /portStatistic.cgi       # Port management
/vlan.cgi, /8021qBasic.cgi, /8021qAdv.cgi  # VLAN configuration
/qos.cgi, /rate_limit.cgi                   # QoS and rate limiting
/port_mirror.cgi                            # Port mirroring
/loop_detect.cgi                            # Loop detection
/igmp.cgi                                   # IGMP snooping
/cable_test.cgi                             # Cable diagnostics
/firmware_update.cgi                        # Firmware upgrade
/factory_default.cgi                        # Factory reset
/register_debug.cgi                         # Direct register R/W (removed in V1.6.0.17)
/produce_burn.cgi                           # Manufacturing/provisioning
```

---

## 5. Security Findings

### 5.1 Hardcoded Debug Credentials

All GS105PE versions (V1.4.0.2 through V1.6.0.17) and GS108Ev3 contain:
```
Admin1NtgrDebugUser
```
This appears to be a Netgear internal debug account that was never removed from production firmware.

### 5.2 NSDP Authentication Secret

GS108Ev2 firmware contains the hardcoded NSDP authentication shared secret:
```
NtgrSmartSwitchRock
```

### 5.3 Register Debug Interface

GS105PE V1.4.0.2 through V1.6.0.10 include `/register_debug.cgi`, a hidden page allowing
direct read/write access to switch registers:
```html
Register Address: 0x1b0c
Bit Mask: 0x30
Value: 3
```
This was removed in V1.6.0.17.

### 5.4 Backdoor Key Mechanism

GS108Ev3 and GS105PE both contain strings suggesting a firmware signing backdoor:
```
Back door key check: success.
%s_V%d.%02d.%02dEN.bin.key
```

---

## 6. Likelihood Assessment: Similar Broadcom/8051 Architecture

### Verdict: Same MCU Architecture, Different Switch Silicon

| Dimension | Similarity | Details |
|-----------|-----------|---------|
| **MCU core** | **Identical** | Both use 8051 — confirmed by IVT, opcodes, IRAM clear |
| **Compiler/toolchain** | **Identical** | Keil C51 on both platforms |
| **Application code** | **High overlap** | Shared source tree (`common/src/`), SAL abstraction |
| **Switch silicon** | **Different** | GS105PE = Realtek RTL8367N/B; GS108E = Broadcom BCM53128 |
| **Firmware format** | **Different** | 0x12345678 header vs UMHD inline header |
| **Code banking** | **Different** | GS105PE linear vs GS108E 64KB banked |
| **TCP/IP stack** | **Related** | LWPS (GS105PE) vs uIP (GS108E) |
| **Register interface** | **Different** | Different SFR maps, different register addresses |

The GS105PE and GS108E share the **same fundamental architecture pattern** — a small 8051 MCU
embedded in the switch SoC, running bare-metal firmware compiled with Keil C51, managing the
switch via memory-mapped registers. Netgear's software team uses a shared codebase with a
Switch Abstraction Layer (SAL) to support both Realtek and Broadcom chips.

However, the GS105PE is **not Broadcom-based**. It uses Realtek RTL8367N/B silicon. Any reverse
engineering work targeting Broadcom BCM53128 register maps (e.g., florolf's work on the GS108Ev2)
will **not directly apply** to the GS105PE's register interface. The 8051 application code and
protocols (NSDP, web UI, DHCP, etc.) are portable across both platforms via the SAL, but the
low-level hardware interaction differs.

### Practical Implications for Reverse Engineering

1. **8051 disassembly tools work on both** — any 8051 disassembler/decompiler can analyze both
   firmware families
2. **Protocol-level work transfers** — NSDP, web API, and management protocols are the same
3. **Register-level work does NOT transfer** — Broadcom RoboSwitch register docs don't apply to
   Realtek RTL8367 registers
4. **The Realtek RTL8367 datasheet** (available from various sources) is the correct reference
   for GS105PE hardware
5. **The SAL layer is the portability boundary** — code above the SAL is shared, code below it
   is chip-specific
