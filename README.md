# Open Recovery Wizard

An original, Windows-first, open-source data recovery application planned around a Rust core, Tauri 2, and a React/TypeScript UI. It is being designed from public specifications, controlled experiments, and clean-room observations of interoperable products.

## Project principles

- Never write to a source device during discovery or recovery.
- Keep acquisition, analysis, preview, and export independently testable.
- Preserve provenance for every result: source range, method, confidence, and validation result.
- Use original source code and visual identity. This repository contains no third-party product binaries, assets, proprietary strings, or license material.
- Do not bypass, emulate, patch, or otherwise defeat activation or other access controls.

## Current state

The repository currently holds a research baseline and original architecture plan, not a recovery engine implementation.

- [Clean-room research charter](docs/00-clean-room-charter.md)
- [Evidence-backed product study](docs/01-observed-reference-architecture.md)
- [Original recovery-engine architecture](docs/02-original-engine-architecture.md)
- [Validation and safety plan](docs/03-validation-and-safety.md)
- [Utility subsystem study](docs/04-utility-subsystems-study.md)
- [Worklog](docs/WORKLOG.md)

## Scope of the first release

Local Windows storage only: physical disks, partitions, removable media, and image files. The core paths are filesystem-aware scan, deleted-file recovery, raw carving, preview, safe export, and a separately sandboxed utility layer for file repair, disk health, and media/document viewing. Network, NAS, Linux, RAID, cloud, and damaged-system rescue remain later milestones.
