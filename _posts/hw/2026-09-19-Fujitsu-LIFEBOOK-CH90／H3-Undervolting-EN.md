---
layout: post
title: "Fujitsu LIFEBOOK CH90/H3 Undervolting"
category: HW
date: 2026-09-19
tags: [Fujitsu, FMV, CH90, setup_var, undervolting, 1255U]
---
> [日本語](./2026-09-19-Fujitsu-LIFEBOOK-CH90／H3-Undervolting-JP.md)
 
---
## Warning
The settings here are based on the FMVC90H3LC (FJNBB7C, BIOS 1.08).
 
On a different laptop, or on a different BIOS version, extract the IFR yourself, check it, and then make changes.
 
Modifying BIOS NVRAM can make the machine unbootable, and recovery needs a BIOS default load or a CMOS clear.
 
If a BIOS upgrade is carried out after the offsets have been modified, there is a chance the offsets get reset.
 
Any problem caused by modifying the offsets is your own responsibility.
 
---
## Environment
Machine: Fujitsu LIFEBOOK CH90/H3 (FMVC90H3LC)<br>
BIOS: 1.08<br>
CPU: Core i7-1255U<br>
Memory: LPDDR5-4800 16GB onboard
 
---
 
## Symptom
The 1255U is a low power CPU, but even so this was performance that was hard to accept, with YouTube 1080p stuttering now and then.
 
Same symptom on both Windows and Linux.
 
---
 
## State before the work
I ran an initial measurement with Cinebench R23 first. The result was as follows.
 
```
Cinebench R23 Multi   3592
```
 
ThrottleStop and HWiNFO readings.
 
```
CPU Package Power   15.069 W (min 14.018 / max 15.924 / avg 15.026)
PL1 Power Limit     Static 15.0 W / Dynamic 15.0 W
PL2 Power Limit     Static 55.0 W / Dynamic 40.0 W
Limit Reasons       PL1 (CORE, RING)
Core Ratio          15.4x
Core VIDs    current 0.757 / min 0.742 / max 0.779 / avg 0.760 V
```
 
PL1 is fixed at 15W, and the HP Spectre x360 13.5, which uses the same i7-1255U, runs at PL1 28W sustained / PL2 44W short burst. After confirming that, I decided to try removing the power limit and undervolting.
 
> From the Notebookcheck spec sheet.
> `Intel Core i7-1255U 10 x 1.2 - 4.7 GHz, 44 W PL2 / Short Burst, 28 W PL1 / Sustained, Alder Lake-U`
> https://www.notebookcheck.com/HP-Spectre-x360-13-5-14t-ef000.647702.0.html
 
There was no power related item on the BIOS setup screen, so I decided to try finding the varStore directly and changing the offsets.
 
---
 
## IFR extraction
Unpacking the BIOS update file `FJNBB7C_108.cap` from the FMV site with UEFIExtract gives InsydeH2O.
The IFR is pulled out of the DXE driver `SetupUtility` (`FE3542FE-C1D3-4EF8-657C-8048606FF670`) with IFRExtractor-RS.
 
I take the four below to be the varstores that can be modified. `SystemConfig`, `FjAdvancedSetup`
and others were there as well, but they are declared as `EFI_IFR_VARSTORE` and were not accessible as UEFI variables.
 
| varstore | GUID |
|---|---|
| Setup | EC87D643-EBA4-4BB5-A1E5-3F3E36B20DA9 |
| SaSetup | 72C5E28C-7783-43A1-8767-FAD73FCCAFA4 |
| CpuSetup | B08F97FF-E6E8-4193-A997-5E9E9B0ADB32 |
| PchSetup | 4570B7F1-ADE8-4943-8DC3-406472842384 |

<br>

---
 
## Modifying offsets with setup_var
I tried making the changes from Windows through `SetFirmwareEnvironmentVariableExW`, but a KMODE_EXCEPTION_NOT_HANDLED blue screen came up.
 
I went through setup_var.efi to make the changes.
 
The USB layout is FAT32 MBR.
 
```
E:\EFI\BOOT\BOOTX64.EFI     pbatard shellx64.efi rename
E:\setup_var.efi            datasone setup_var.efi v0.3.1
```
 
Turn Secure Boot off in the BIOS and boot with F12.
 
There are three things to watch for.
 
- Variable names are case sensitive. `Cpusetup` gives Not Found, `CpuSetup` works
- The number in parentheses is the byte count. Putting `(0)` on a 2 byte field gives `Specified value to write is larger than specified size 0 bytes`
- When several variables share a name you need the ID. There are two of `Setup`, so it has to be written as `Setup(0x1)`

All three can fail silently or write a wrong value, so after a write command you have to check whether the write went through properly.
 
---
### Removing the power limit (failed)
<details markdown="1">
<summary>What I tried</summary>

#### Unlocking the MSR
```
setup_var.efi CpuSetup:0x30=0x0
setup_var.efi CpuSetup:0x30
```
After a reboot the padlock was gone from the ThrottleStop TPL window, so I judged that it had applied.
 
But raising MSR PL1 to 28W with ThrottleStop did not move the actual power draw from 14.9W. HWiNFO was in the state below as well.
 
```
PL1 Power Limit     Static 28.0 W / Dynamic 15.0 W
PL2 Power Limit     Static 55.0 W / Dynamic 40.0 W
```
Only the MSR side changed and the MMIO side stayed as it was. I had to find the path that programs MMIO separately.
 
#### MMIO and cTDP
The `Config TDP Configurations` form in the IFR has Text items that show the MMIO power limits, and Custom Settings was attached right below them.
 
```
Text: "Power Limit 1"  Help: "Power Limit 1 values from MMIO"
Text: "Power Limit 2"  Help: "Power Limit 2 values from MMIO"
---- Custom Settings Nominal ----
Numeric: "Power Limit 1"  -> CpuSetup:0x5B
Numeric: "Power Limit 2"  -> CpuSetup:0x5F
```
 
Reading `0x5B` and `0x5F` gave 0 for both. I judged that when the Custom values are empty the cTDP Nominal level programs the SKU default of 15W, and put values in.
 
```
setup_var.efi CpuSetup:0x227=0x1
setup_var.efi CpuSetup:0x5B(4)=0x6D60
setup_var.efi CpuSetup:0x5F(4)=0xD6D8
setup_var.efi CpuSetup:0x63=0x38
setup_var.efi CpuSetup:0x227
setup_var.efi CpuSetup:0x5B(4)
setup_var.efi CpuSetup:0x5F(4)
setup_var.efi CpuSetup:0x63
```
 
| offset | value | item |
|---|---|---|
| 0x227 | 1 | Enable Configurable TDP |
| 0x5B | 0x6D60 | Custom Nominal PL1 = 28,000 mW |
| 0x5F | 0xD6D8 | Custom Nominal PL2 = 55,000 mW |
| 0x63 | 0x38 | Power Limit 1 Time Window = 56 s |
 
After a reboot both the MSR and the MMIO side changed. It applies from the BIOS alone, without ThrottleStop.
 
```
PL1 Power Limit     Static 28.0 W / Dynamic 28.0 W
PL2 Power Limit     Static 55.0 W / Dynamic 55.0 W
```
 
#### Effective power stayed the same...
The registers became 28W, but sustained power under load stayed at 15W. Limit Reasons was still PL1.
 
I checked whether some other limit was in play. For PSYS, `CpuSetup:0x32` is 60,000 mW while the highest Total System Power across all the measurements was 45W, and for IccMax, ThrottleStop FIVR shows 80A while the highest VR VCC Current was 38A. Moving PROCHOT to 95, 100 and 96 degrees with the Tcc Activation Offset gave sustained power of 14.895W, 14.846W and 14.925W, effectively the same. Turning Intel DTT off with `Setup(0x1):0x6B1=0x0` made no difference either.
 
My own read is that this is a problem caused by being caught on a Fujitsu gate.
 
```
SuppressIf QuestionId 0x10E == 0     ->  SystemConfig:0x11D  "Power Limit Override:"
GrayOutIf  QuestionId 0x1053 == 0    ->  SystemConfig:0x1A9  (OverClocking Feature)
```
 
`SystemConfig` is not accessible as a UEFI variable, so there is no way to touch that side. I treated removing the power limit as finished here and stopped.
</details>
---
 
### Undervolting
With power fixed at 15W, I judged there was nothing left but to turn toward lowering V.
 
#### Unlocking
```
setup_var.efi CpuSetup:0x1D9=0x1
setup_var.efi CpuSetup:0x10E=0x0
setup_var.efi CpuSetup:0x1D9
setup_var.efi CpuSetup:0x10E
```
 
| offset | value | item |
|---|---|---|
| 0x1D9 | 1 | OverClocking Feature |
| 0x10E | 0 | Overclocking Lock |
 
#### Domains

| domain | Mode | Prefix | Offset | bytes |
|---|---|---|---|---|
| P-core | CpuSetup:0x1DD | CpuSetup:0x1E2 | CpuSetup:0x1E0 | 2 |
| E-core L2 | CpuSetup:0x2AF | CpuSetup:0x2B4 | CpuSetup:0x2B2 | 2 |
| Ring | CpuSetup:0x1E9 | CpuSetup:0x1EE | CpuSetup:0x1EC | 2 |
| Uncore | CpuSetup:0x2DE | SaSetup:0x261 | SaSetup:0x25F | 2 |
 
For Mode, 0 is Adaptive and 1 is Override. To use the offset method it has to be left at 0. For Prefix, 0 is plus and 1 is minus. For Offset, put the mV value in as hex just as it is.
 
`CpuSetup:0x25F` is VF Point 14 Offset Prefix, so it only shares the offset number with Uncore and is a completely different item. Check the varstore before writing.
 
#### Checking that it applied
On this machine the `Voltage Offsets` item in HWiNFO always shows 0.000 V for every domain. ThrottleStop FIVR also stays `Not Available`. I judge that both are because runtime mailbox reads are blocked, and that it has nothing to do with whether the setting applied.
 
I did the actual check with the `Core VIDs` average under load.
 
| P-core setting | Core VIDs avg | R23 Multi |
|---|---|---|
| -50 mV | 0.905 V | 3990 |
| -100 mV | 0.856 V | 4220 |
 
A 50mV change in the setting against a measured VID change of 49mV, which is close enough to match.
 
#### Finding the voltage values
 
I lowered one domain at a time in 20 to 50mV steps, rebooted, and ran Cinebench R23 multi core.
 
Going too low showed up as a performance drop rather than a crash.
The requested clock stays the same while only the effective clock falls, and power and temperature sink along with it.
I see it as clock stretching or CEP stepping in.
 
These are the measurements from lowering E-core L2 from -50mV to -80mV.
 
| metric | -50 mV | -80 mV |
|---|---|---|
| R23 Multi | 4303 | 3599 |
| Package Power max | 28.7 W | 17.8 W |
| VR VCC Current max | 38.1 A | 18.75 A |
| CPU Package temperature max | 98 C | 76 C |
| Core Effective Clocks max | 2849 MHz | 1741 MHz |
| Core Clocks max (requested) | 4091 MHz | 4090 MHz |
 
I set the decision criteria on this pattern. If the maximum temperature under load was below 88 degrees, or the maximum VR current below 25A, I took it as having gone too low and rolled it back. You can tell from the screen alone before measuring a score.
 
When measuring, the Average column in HWiNFO depends on the measurement window. Capturing after the benchmark ends mixes in the idle stretch and makes comparison impossible. It is better to check first whether the `Core Usage` average in the captured screen is 80% or above.
 
#### Results per domain
 
| setting | R23 Multi |
|---|---|
| no undervolt | 3592 / 3600 / 3938 |
| P -50 | 3990 |
| P -100 | 4220 |
| P -100 / E-L2 -50 | 4303 / 4260 |
| P -100 / E-L2 -80 | 3599 |
| P -100 / E-L2 -50 / Ring -50 | 4440 |
| P -100 / E-L2 -50 / Ring -70 | 4439 |
| P -120 / E-L2 -50 / Ring -50 | 4229 |
 
| domain | best value | limit |
|---|---|---|
| P-core | -100 mV | score drops at -120mV |
| E-core L2 | -50 mV | collapses at -80mV |
| Ring | -50 mV | no gain at -70mV |
| Uncore | not applied | not attempted |
 
Ring gave 4439 at -70mV against 4440 at -50mV, and since the difference was marginal I rolled it back to -50mV.
 
---
 
## Final settings
 
```
# power limit (cTDP / MSR lock) - no actual effect, for reference only
setup_var.efi CpuSetup:0x227=0x1
setup_var.efi CpuSetup:0x5B(4)=0x6D60
setup_var.efi CpuSetup:0x5F(4)=0xD6D8
setup_var.efi CpuSetup:0x63=0x38
setup_var.efi CpuSetup:0x30=0x0
 
# thermal - PROCHOT 95C to 96C
setup_var.efi CpuSetup:0x7F=0x4
 
# Intel DTT disable
setup_var.efi Setup(0x1):0x6B1=0x0
 
# undervolt - unlock
setup_var.efi CpuSetup:0x1D9=0x1
setup_var.efi CpuSetup:0x10E=0x0
 
# undervolt - P-core -100mV
setup_var.efi CpuSetup:0x1DD=0x0
setup_var.efi CpuSetup:0x1E2=0x1
setup_var.efi CpuSetup:0x1E0(2)=0x64
 
# undervolt - E-core L2 -50mV
setup_var.efi CpuSetup:0x2AF=0x0
setup_var.efi CpuSetup:0x2B4=0x1
setup_var.efi CpuSetup:0x2B2(2)=0x32
 
# undervolt - Ring -50mV
setup_var.efi CpuSetup:0x1E9=0x0
setup_var.efi CpuSetup:0x1EE=0x1
setup_var.efi CpuSetup:0x1EC(2)=0x32
 
# verify applied
setup_var.efi CpuSetup:0x227
setup_var.efi CpuSetup:0x5B(4)
setup_var.efi CpuSetup:0x5F(4)
setup_var.efi CpuSetup:0x63
setup_var.efi CpuSetup:0x30
setup_var.efi CpuSetup:0x7F
setup_var.efi Setup(0x1):0x6B1
setup_var.efi CpuSetup:0x1D9
setup_var.efi CpuSetup:0x10E
setup_var.efi CpuSetup:0x1DD
setup_var.efi CpuSetup:0x1E2
setup_var.efi CpuSetup:0x1E0(2)
setup_var.efi CpuSetup:0x2AF
setup_var.efi CpuSetup:0x2B4
setup_var.efi CpuSetup:0x2B2(2)
setup_var.efi CpuSetup:0x1E9
setup_var.efi CpuSetup:0x1EE
setup_var.efi CpuSetup:0x1EC(2)
```
 
`Setup(0x1):0x6B1` disables Intel DTT. It had no effect on performance, but every measurement was taken in this state, so I am writing it down together with the rest.
 
`CpuSetup:0x7F` is the Tcc Activation Offset.
I judged there was no problem with it personally, so I raised the factory value of 5 (PROCHOT 95 degrees) by a single step to 4 (96 degrees).
 
It must not be set to 0. The `_HOT` trip in the DSDT thermal zone is 98 degrees and `_CRT` is 99 degrees, so I confirmed that raising PROCHOT to 100 degrees makes Windows go into S4 hibernation the moment it passes 98 degrees.
 
---
 
## Result
 
| | value |
|---|---|
| Cinebench R23 Multi (before the work) | 3592 |
| Cinebench R23 Multi (final) | 4440 |
| improvement | 23.6% |
| sustained power | 15W (unchanged) |
 
Sustained power is unchanged, so I judge the whole of the gain to be an efficiency improvement from undervolting.
 
The minimum for the i7-1255U in the Notebookcheck database is 5269 points, so when I have time I intend to look for somewhat more suitable values...
 
---
 
## What I could not do
 
The power limit is blocked by the Fujitsu `SystemConfig` gate. It is not accessible as a UEFI variable, and this board has Boot Guard and BIOS Guard, so an unsigned BIOS image will not flash.
 
Fan speed control is the same. The ACPI fan device in the DSDT has `_FST` but no `_FSL`, and the thermal zone has no `_AC0` or `_AL0` either. The fan mode switch is done through one bit in EC RAM 0x65 (`FSLM`) and EC command 0x79, and there are only two states, Normal and Silent. The fan trip point items in the BIOS setup (`Setup:0x697` to `0x6A0`) did not take effect on this laptop.
 
