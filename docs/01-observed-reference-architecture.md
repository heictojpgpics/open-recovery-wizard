# Evidence-backed reference study

## Evidence identity

All statements are based on static, read-only analysis of the supplied payload on 2026-10-15. The binaries were not executed.

| Sample | SHA-256 | Result |
| --- | --- | --- |
| `EaseUS-Data-Recovery-Wizard-Part-1.zip` | `af2d44077d53d3348ae6b57863a2cd31c6dcadb2b616af7a1b89e3c287e90a14` | ZIP integrity test passed |
| `EaseUS-Data-Recovery-Wizard-Part-2.zip` | `91aa2dab86e3faa40306a36d3ccdff3e1c3b9df8d1d22a2dbb845675eb2c1e48` | ZIP integrity test passed |
| macOS DMG | `6ccb507ea097eeb41b39218602f22a144b65c02b27e8865134d9ebcdbfe31270` | downloaded; not inspected in this baseline |
| Windows NSIS installer | `31469c17128d37c60d3bc1228b1bbc8f793a9d973bae42b16f25a8ce338b7e86` | identified as NSIS v3.03; not executed |

The two ZIP archives are complementary deployment trees, not split segments: extracting both into one directory yielded 1,470 files and approximately 674 MiB.

## Observed technology profile

**Observed.** The principal executable and recovery modules examined are 64-bit native Windows PE files. The installer is a 32-bit NSIS package requesting administrative privilege. `DRWUI.exe` imports Qt 5 Core/GUI/Network/SVG/Widgets/WinExtras plus Windows setup, shell, crypto, networking, trust, and MSI APIs. Qt platform, image-format, and style plugins are bundled. This supports the narrow conclusion that the observed UI is native C++/Qt rather than .NET or Electron.

**Observed.** Main version metadata identifies `21.0.0.0`; packaging configuration identifies `free` and `RE-DRW-FREE-WIN`. Configuration lists 23 UI languages and high-DPI operation. These product facts are not used as a UI template.

## Observed recovery subsystems

| Subsystem | Evidence | Confidence | Clean-room requirement |
| --- | --- | --- | --- |
| Device/partition layer | Dedicated device, partition, manager, boot, volume and recovery modules; exported storage-device operations include `DeviceIoControl`. | High | Separate read-only block-device abstraction with range-checked reads and a partition map. |
| Filesystem-aware scan | Standalone x64 modules: NTFS, FAT, exFAT, EXT4, Btrfs, ReFS, ZFS and HFS+. Each exposes a distinct quick and standard/complete searcher constructor. | High | Per-filesystem parsers behind a stable scanner trait; first release supports NTFS, FAT32, exFAT and EXT4. |
| Raw carving | `RAWSearchDLL.dll` exports raw-searcher factory and analyser-list/map functions. JSON configuration contains 139 signature definitions, 46 with tail constraints, plus size bounds. | High | Declarative signatures plus a bounded streaming carver with header, footer, size, and validator stages. |
| Quick then deep scan | UI resource strings distinguish Quick Scan and Advanced Scan; a string describes advanced scanning as sector-by-sector. | High | Fast metadata pass followed by opt-in resumable deep scan. |
| Resume scan | `SaveScan` directory, `.rsf` records, and UI strings for loading/restoring prior session. | High | Versioned, checksummed checkpoints bound to device identity and geometry. |
| Preview | UI strings and image/PDF/office/video viewer components show a preview flow. | High | Out-of-process preview adapters using copied bytes or a range-limited virtual reader. |
| Recovery orchestration | UI strings name auto-recovery; recovery module contains single/multiple/pending operation terminology. | Medium-high | Manifest-driven cancellable export queue with deterministic collision policy. |
| BitLocker awareness | Module/interface names and UI labels mention status and unlocking. | Medium-high | Detect and label encrypted volumes; no decryption claim without independent tests. |
| Partition recovery | Dedicated partition-recovery directory and strings referencing MBR/GPT, volume and partition chains. | High | Later module that searches valid metadata and scores candidates without mutation. |

## What the evidence does and does not prove

The evidence proves deployed module boundaries, exports, configuration-driven type coverage, and visible workflow vocabulary. It does **not** prove internal algorithms, data structures, scan order, proprietary heuristics, license logic, or correctness claims. Those must be independently specified and validated.

## Export-level interface observations

The scanner DLLs expose factory names of the form `EUCreate<filesystem>QuickSearcher` and `EUCreate<filesystem>Searcher` (Btrfs uses `ComplateSearcher`). The raw module exports `EUCreateRAWSearcher`, `EUGetAnalyserList`, `EUGetAnalyserMap`, and `EUGetAnalyserMultimap`. This indicates a plug-in/factory boundary, not a reusable public specification. Our own implementation will use a separately designed Rust trait and data model rather than these interfaces.

## Functional flow inferred from evidence

The components support this behavioral model, which remains an inference: select a device or partition → metadata-oriented quick scan → sector-level scan and/or raw carver → filter/search/preview results → recover to another destination → optionally resume prior scan state. A safe original implementation makes every stage cancellable and never treats the inference as exact.

## Scope assessment

The payload also includes file repair, disk health, cloud/remote configuration, NAS, media viewing, office/PDF viewing, and other utilities. These are excluded from the initial implementation.

## Evidence index

Local ignored analysis outputs: `reports/file_inventory.tsv`, `reports/pe_architecture.txt`, `reports/core_imports.txt`, `reports/configuration_extracts.txt`, `reports/recovery_strings.txt`, and `reports/provenance_and_runtime_strings.txt`. They remain local because they derive directly from proprietary payloads.
