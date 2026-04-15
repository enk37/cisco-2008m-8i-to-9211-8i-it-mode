# Cisco SAS2008M-8i to 9211-8i IT Mode Cross-Flash

This repository contains a publishable runbook from a real lab cross-flash effort involving:

- cross-flashing a Cisco `UCSC RAID SAS 2008M-8i` / `RC460-PL002`-class controller into `9211-8i` IT mode
- documenting the exact Linux `lsirec` + `lsiutil` workflow that worked
- recording the platform-specific pitfalls that mattered on Cisco UCS hardware

Main document:
- [`IT_MODE_CROSSFLASH_RUNBOOK.md`](IT_MODE_CROSSFLASH_RUNBOOK.md)

Bundled source trees:
- `lsirec/` contains the original upstream `lsirec` source used in the workflow
- `lsiutil/` contains the original upstream `lsiutil` source used in the workflow

Scope note:
- this repository is about the controller cross-flash only
- it does not attempt to document Samsung SSD recovery beyond noting that the HBA conversion itself was successful
