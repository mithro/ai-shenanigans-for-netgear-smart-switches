# Netgear Smart Switch Firmware Collection

All firmware files downloaded from Netgear's official support site (kb.netgear.com / downloads.netgear.com).

## GS105PE - 5-Port Gigabit Ethernet Smart Managed Plus Switch (PoE Pass-Thru)

All 10 published firmware versions:

| Version | Date | Size (bin) | KB Article |
|---------|------|-----------|------------|
| V1.4.0.2 | 2015-06-10 | 699,788 | [KB 29728](https://kb.netgear.com/29728/GS105PE-Firmware-Version-1-4-0-2) |
| V1.4.0.6 | 2015-12-28 | 706,668 | [KB 30222](https://kb.netgear.com/app/answers/detail/a_id/30222) |
| V1.4.0.9 | 2016-05-30 | 726,456 | [KB 30939](https://kb.netgear.com/app/answers/detail/a_id/30939) |
| V1.5.0.4 | 2017-02-09 | 745,128 | [KB 38422](https://kb.netgear.com/000038422/GS105PE-Firmware-Version-1-5-0-4) |
| V1.5.0.5 | 2017-12-22 | 744,180 | [KB 53860](https://kb.netgear.com/000053860/GS105PE-Firmware-Version-1-5-0-5) |
| V1.6.0.3 | 2018-09-12 | 809,220 | [KB 60281](https://kb.netgear.com/000060281/GS105PE-Firmware-Version-1-6-0-3) |
| V1.6.0.4 | 2019-02-18 | 817,728 | [KB 60850](https://kb.netgear.com/000060850/GS105PE-Firmware-Version-1-6-0-4) |
| V1.6.0.6 | 2021-09-03 | 819,464 | [KB 61257](https://kb.netgear.com/000061257/GS105PE-Firmware-Version-1-6-0-6) |
| V1.6.0.10 | 2021-09-03 | 878,924 | [KB 63990](https://kb.netgear.com/000063990/GS105PE-Firmware-Version-1-6-0-10) |
| V1.6.0.17 | 2023-04-18 | 676,288 | [KB 65637](https://kb.netgear.com/000065637/GS105PE-Firmware-Version-1-6-0-17) |

V1.5.0.4 also includes bootloader binaries (`GS105PE_loader_V1.4.0.5.bin` and `GS105PE_loader_V1.4.0.5-VB.bin`).

## GS108Ev2 - 8-Port Gigabit Ethernet Smart Managed Plus Switch (for comparison)

| Version | Date | Size (bin) | KB Article |
|---------|------|-----------|------------|
| V1.00.12 | 2013-06-11 | 262,144 | [KB 23769](https://kb.netgear.com/23769/GS108Ev2-Firmware-Version-1-00-12) |

## GS108Ev3 - 8-Port Gigabit Ethernet Smart Managed Plus Switch (for comparison)

| Version | Date | Size (bin) | KB Article |
|---------|------|-----------|------------|
| V2.06.24EN | 2023-04-18 | 720,896 | [KB 65636](https://kb.netgear.com/000065636/GS108Ev3-Firmware-Version-2-06-24) |

## Directory Structure

```
firmware/
├── GS105PE/
│   ├── GS105PE_V1.4.0.2/          # Extracted firmware + release notes
│   ├── GS105PE_V1.4.0.2.zip       # Original download
│   ├── ...                         # (10 versions total)
│   └── GS105PE_V1.6.0.17/
├── GS108Ev2/
│   ├── GS108EV2_V1.00.12/
│   └── GS108EV2_V1.00.12.zip
├── GS108Ev3/
│   ├── GS108Ev3_V2.06.24EN/
│   └── GS108Ev3_V2.06.24EN.zip
└── README.md
```
