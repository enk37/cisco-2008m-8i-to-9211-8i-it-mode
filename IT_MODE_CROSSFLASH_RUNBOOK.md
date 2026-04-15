# Cross-Flashing a Cisco SAS2008M-8i Mezzanine Card to 9211-8i IT Mode

This write-up captures a workflow that was actually validated on a Cisco SAS2008-based mezzanine/RAID controller and converted it into a working `9211-8i`-style IT-mode HBA under Linux.

It documents what worked, what failed, and the specific traps that mattered for the controller conversion itself.

## Validated Test Platform

The validated run documented here used:
- server: `Cisco UCS C220 M3`
- operating system: `Xubuntu 25.10 minimal`
- boot medium: USB stick

That host detail matters because two environment-specific lessons came out of it:
- FreeDOS `sas2flsh.exe` was not viable due to PAL/BIOS32 issues on this platform
- UEFI `sas2flash.efi` also did not provide a viable flash path on this platform
- GRUB/kernel-arg IOMMU disable caused repeated `initramfs` failures, while disabling VT-d in BIOS worked

## Scope

This runbook is for:
- SAS2008-based LSI/Cisco RAID cards
- especially Cisco `UCSC RAID SAS 2008M-8i` / `RC460-PL002`-class hardware
- Linux-based recovery/cross-flash using `lsirec` and `lsiutil`

This is **not** for:
- SAS3008/SAS3 cards
- generic “flash any LSI card” use
- SSD firmware recovery workflows

## Outcome

Successful part:
- the controller was converted from Cisco MegaRAID personality to a working `9211-8i`-style IT-mode HBA
- Linux saw it as:
  - PCI ID `1000:0072`
  - subsystem `1000:3020`
  - driver `mpt3sas`

## Risk statement

This can brick the HBA.

Do not do this on a controller that is still needed in production.

Back up everything you can first:
- SBR
- complete flash
- current PCI identity
- SAS WWID

## Why this path was needed

Conventional paths were not reliable on this platform:
- FreeDOS `sas2flsh.exe` failed with PAL/BIOS32 initialization issues
- UEFI `sas2flash.efi` also failed to provide a usable cross-flash path on this host
- GRUB kernel-arg IOMMU disable caused repeated `initramfs` failures on this host

What finally worked:
- disable VT-d in BIOS
- use Linux `lsirec`
- use `lsiutil` after `hostboot`

## Files you need

### 1. `lsirec`

Source:
- [marcan/lsirec](https://github.com/marcan/lsirec)

Useful files in that repo:
- `lsirec.c`
- `sbrtool.py`
- `sample_sbr/sbr_sas9211-8i_itir.cfg`

### 2. `lsiutil`

The `lsirec` README points to an `lsiutil` source location:
- [exactassembly/meta-xa-stm lsiutil files](https://github.com/exactassembly/meta-xa-stm/blob/master/recipes-support/lsiutil/files/)

Build `lsiutil` locally on Linux before starting the risky steps.

One workable approach is:

```bash
mkdir -p ~/sas2flash/lsiutil-src
cd ~/sas2flash/lsiutil-src
curl -L -o lsiutil-1.72.tar.gz \
  https://github.com/exactassembly/meta-xa-stm/raw/master/recipes-support/lsiutil/files/lsiutil-1.72.tar.gz
tar -xzf lsiutil-1.72.tar.gz
cd lsiutil
make -f Makefile_Linux
```

Expected output binary:

```text
./lsiutil
```

In the validated run, the built binary lived at:

```text
/home/enk/sas2flash/lsiutil-src/lsiutil/lsiutil
```

Notes:
- this is old code and may emit compiler warnings
- the warnings did not prevent the validated cross-flash workflow
- do not install it system-wide unless you actually want that

### 3. `2118it.bin`

You need a known-good `9211-8i` IT firmware image.

In the validated run, the working file was:
- `2118it.bin`
- size: `722708`
- sha256: `9c35132531e515b3aa066ba2940f4c11bc953eb16eba61d7a34620eb290d1cd2`

Validated source used in this project:
- Broadcom package: [`9211-8i_Package_P20_IR_IT_Firmware_BIOS_for_MSDOS_Windows`](https://docs.broadcom.com/docs/12350530)

That package title is important because it gives readers a concrete upstream source instead of an unspecified mirrored firmware blob.

Other possible sources:
- extracted OEM preboot/firmware bundles if you trust the provenance

### 4. Optional BIOS / option ROM

Optional only if you need to boot from the HBA:
- `MPTSAS2.ROM`

Not required for pure IT-mode data access.

## Host prerequisites

### Disable VT-d in BIOS

Do this in BIOS.

Do **not** assume kernel-arg IOMMU disable is safe. On the validated host, GRUB/kernel-arg IOMMU disable repeatedly caused `initramfs` failures.

### Hugepages

Before `hostboot`:
```bash
echo 16 > /proc/sys/vm/nr_hugepages
```

### Verify IOMMU is inactive

```bash
find /sys/kernel/iommu_groups -mindepth 1 -maxdepth 1 | wc -l
```

Expected:
```text
0
```

## Confirm the controller first

Verify the active controller is actually SAS2008-based:

```bash
lspci -nnk | grep -A3 -i 'Serial Attached SCSI'
```

In the validated case, the starting identity was:
- `1000:0073`
- subsystem `1137:00b0`
- `megaraid_sas`

## Back up the existing SBR

```bash
./lsirec 0000:81:00.0 readsbr sbr_backup.bin
python3 sbrtool.py parse sbr_backup.bin sbr.cfg
```

Back up the resulting parsed config too.

## Build a 9211-style SBR

Use the sample:
- `sample_sbr/sbr_sas9211-8i_itir.cfg`

In the validated conversion, the key target fields were:

```text
PCIVID = 0x1000
PCIPID = 0x0072
HwConfig = 0x0107
SubsysVID = 0x1000
SubsysPID = 0x3020
Interface = 0x00
```

Then build it:

```bash
python3 sbrtool.py build sbr.cfg sbr_new.bin
```

## Write the new SBR

```bash
./lsirec 0000:81:00.0 writesbr sbr_new.bin
```

This does not fully take effect until the next stage.

## Host-boot the IT firmware

```bash
./lsirec 0000:81:00.0 hostboot 2118it.bin
```

Expected shape of success:
- controller reaches `IOC is READY`
- host boot successful

## Rescan the PCI device

```bash
./lsirec 0000:81:00.0 rescan
```

At this point, Linux should switch from `megaraid_sas` to `mpt3sas`.

Check:

```bash
lspci -nnk | grep -A3 -i 'Serial Attached SCSI'
```

## Make a complete flash backup before destructive writes

Run:

```bash
lsiutil -e
```

Then in the menu:
- select the adapter
- `46 -> 5` Complete (all sections)

Save that image somewhere safe.

In the validated run, the complete flash backup was:
- `16777216` bytes
- sha256 `dffab0dd410657cb30c7b2fd7f2586a4792e8472e58882b3532581f8111a646d`

## Erase flash and persistent manufacturing pages

Still in `lsiutil`:
- `33 -> 3` FLASH
- `33 -> 8` Persistent manufacturing config pages

You need both.

## Download the firmware in `lsiutil`

In `lsiutil`:
- `2` Download firmware

Point it at `2118it.bin`.

This was the decisive permanent-flash step.

## Reset and rescan

```bash
./lsirec 0000:81:00.0 reset
./lsirec 0000:81:00.0 rescan
```

Important nuance:
- the first reset after permanent flash may complain or land in fault temporarily
- in the validated run, one early reset faulted once, but a later rescan recovered it and the controller came back normally

Check:

```bash
./lsirec 0000:81:00.0 info
```

What you want:
- `IOC is OPERATIONAL`

## Program a unique SAS WWID

Do this only after the controller is otherwise stable.

Use:
- `lsiutil -> 18 Change SAS WWID`

Then:

```bash
./lsirec 0000:81:00.0 reset
./lsirec 0000:81:00.0 rescan
```

### Critical warning about SAS WWIDs

Do **not** blindly trust a CIMC-reported SAS address and copy it into the HBA.

In the validated project, an initially programmed value matched one attached Samsung SSD on the live SAS bus, creating a real SAS-address collision.

The fix was:
- choose a new unique HBA WWID
- then verify it does not match any end-device SAS address

In the validated project, the final unique HBA WWID used was:
- `500605b0c7a1d24e`

## Final verification

Verify PCI identity:

```bash
lspci -nnk | grep -A3 -i 'Serial Attached SCSI'
```

Expected:
- `1000:0072`
- subsystem `1000:3020`
- `mpt3sas`

Verify HBA health:

```bash
./lsirec 0000:81:00.0 info
```

Expected:
- `IOC is OPERATIONAL`

Verify HBA WWID and end-device addresses:

```bash
for f in /sys/class/scsi_host/host*/host_sas_address; do echo "$f: $(cat "$f")"; done
for f in /sys/class/sas_device/end_device-*/sas_address; do echo "$f: $(cat "$f")"; done
```

Verify disk enumeration:

```bash
lsscsi -g
lsblk
```

## What did not work for the controller path

These paths were tried and rejected in the validated project:

### 1. FreeDOS `sas2flsh.exe`

Problem:
- PAL / BIOS32 initialization failure on this host

Conclusion:
- not a viable primary path here

### 2. UEFI `sas2flash.efi`

Problem:
- also not a viable flash path on this host

Conclusion:
- the successful route was Linux `lsirec` + `lsiutil`, not DOS or EFI `sas2flash`

### 3. GRUB/kernel-arg IOMMU disable

Problem:
- repeated drops into `initramfs`

Conclusion:
- BIOS VT-d disable was the correct approach on this host

## Summary

If your goal is:
- **cross-flash the Cisco SAS2008M-8i controller into IT mode**, this workflow is proven

This controller-focused runbook is worth keeping as a standalone skill because the HBA conversion worked end to end.
