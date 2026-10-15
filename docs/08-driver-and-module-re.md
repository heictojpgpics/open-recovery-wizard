# Deep RE Findings: Kernel Drivers and Search Modules

Evidence tier: static analysis only (Ghidra 12.1.4 headless decompilation,
PE import/export tables, shipped configuration files, RTTI class names,
embedded PDB paths). No vendor code, strings, or assets are reproduced in
this repository; everything below is a functional description written from
scratch so we can build our own original implementation.

Evidence artifacts live outside this repo (local, gitignored) under
`re-evidence/`: `decomp/<module>.c` (full decompilation), `reports/`
(function inventories, PE import tables), plus per-batch analyzer logs.

## 1. Payload inventory (1470 files, 674 MB)

The product ships as a Qt 5 (x64) application with:

| Layer | Components |
|---|---|
| UI | DRWUI.exe (30 MB, Qt), DRW.exe (launcher, 127 KB), DRWReport.exe |
| Service | DRWService.exe (40 KB) |
| Per-FS engines | NTFSSearchDLL, FATSearchDLL, exFATSearchDLL, EXT4SearchDLL, HFSPlusSearchDLL, BtrfsSearchDLL, ReFSSearchDLL, ZfsSearchDLL, RAWSearchDLL (signature carver) |
| Repair/view | EUAdobeRepair, EUArchiveRepair, EUImgRepair, EURawImg, EUOfficeViewer, PDFViewer (pdfium), EUHwpViewer, EUVideoRepair.dll, EUImageEnhance.dll |
| Partition recovery | PartitionRecovery/ bundle (own Qt applet + drivers) |
| Boot media | BUILDPE/ WinPE tree with its own driver set |
| Cloud/NAS | cloud_session.dll, EuICloudRecovery.dll, EuShare/ (with EuMrx.sys mini-redirector driver), NasConfig |
| Carving data | raw_signs.json (see 4) |

Internal project name (from embedded PDB paths, e.g.
`E:\DRW\_drw_main\DRWIntelligentScan\bin\Release\x64\EXT4SearchDLL.pdb`):
the engine DLLs are built from a project called **DRWIntelligentScan**.

## 2. Kernel driver map

Seven drivers ship in the payload. Hashing first: `ebrntdrv.sys`
(BUILDPE/system32) is byte-identical to `eprdtdrv_win7older.sys`
(PartitionRecovery), and `eprdtdrv.sys` differs only in name/version
resources. So there are five distinct code bases:

| Driver | Size | Distinct imports (ntoskrnl) | Function count | Assessed purpose |
|---|---|---|---|---|
| eprdtdrv.sys / ebrntdrv.sys | 27 KB | IoGetLowerDeviceObject, IoBuildDeviceIoControlRequest, IoCreateDevice, IoCreateSymbolicLink/IoDeleteSymbolicLink, ExAllocatePoolWithTag | 39 | Volume filter + raw-device provider. Debug tag `[EPMNtDrv]` (shared lineage with the partition product). Formats device names `\Device\Harddisk%u\Partition0`, `\Device\HarddiskVolume%u`, and creates raw recovery devices `\Device\Harddisk%u\DR%u` exposed to the app via a symbolic link; uses `wcsstr(..., L"\Device\Harddisk")` to locate stack targets. |
| epmdkdrv.sys | 27 KB | same set as eprdtdrv (identical shape, 39 funcs) | 39 | Same code base re-badged for the boot (WinPE) tree. |
| EUEDKEPM.sys | 19 KB | same set, small | 36 | Minimal variant; "EPM" lineage, disk-enumeration filter. |
| EUDCPEPM.sys | 72 KB | + ZwQueryValueKey, registry/value reads | 177 | Largest of the set; reads configuration from the registry, expanded dispatch logic. Still a filter/raw-device driver, no filesystem parsing. |
| EuMrx.sys | 216 KB | ksecdd.sys GetSecurityUserInfo, MmMapLockedPagesSpecifyCache, ZwCreateFile, cache manager interplay | 436 | Windows mini-redirector driver ("Mrx" = mini-redirector class). This is the NAS/SMB leg: a filesystem redirector so network shares can be browsed/recovered. Not a disk filter. |

Key conclusion: **none of the drivers contain recovery logic.** All
parsing happens in user mode (the *SearchDLL modules). The drivers exist
to (a) obtain exclusive raw access to busy volumes by attaching into the
volume stack, (b) expose stable raw device names, and (c) for NAS, to
mount network shares as filesystems.

### Our decision (kernel drivers)

We do not ship a kernel driver:

- Raw reads of `\\.\PhysicalDriveX` and volume locking
  (`FSCTL_DISMOUNT` + `FSCTL_LOCK_VOLUME`) are sufficient for the
  Windows-local product and require no driver, no WHQL, no test signing.
- Our `dw-disk` crate already implements strict `ReadAt` over file-backed
  images and physical devices; the remaining gap (live-volume exclusive
  access) is a user-mode `FSCTL` sequence, documented as engine work.
- A driver is justified later if we add boot-time recovery (WinPE) or
  in-fs filtering; that decision is recorded with this evidence.

## 3. EXT4SearchDLL (first engine module decoded)

- Exports a C++ class-factory surface: `EUCreateEXT4Searcher`,
  `EUCreateEXT4QuickSearcher`. RTTI names the classes
  `CEXT4CompleteSearch`, `CEXT4QuickSearch` (implements
  `IQuickFileSystemSearchController`), `Ext4Volume`, `CExt4DataRunList`,
  `CEXT4FileProperties`, plus `EXT4CatchBufferObj` /
  `CEXT4CatchBufferDeviceProxy` (a buffered device proxy used during
  scanning).
- Quick search = metadata walk of the live filesystem (directory
  traversal; log strings `TraverseDir`, `SubDirFDT`).
- Complete search additionally traverses the ext4 **journal**
  (`TraverseJournalDir`, `GotJournalDir: id, parentId, sector`): deleted
  directory entries are recovered from journal blocks. That is their
  deep-recovery edge for ext4; the on-disk format itself (superblock at
  1024, `0xEF53` magic, group descriptors, inode tables, extent trees)
  is handled exactly per the public spec, including block-size
  validation `1024 << s_log_block_size` and volume geometry sanity
  checks before any inode is touched.
- Our engine implications: (1) mirror the quick/deep split we already
  have (quick = FS metadata walk; deep = unallocated-region carving);
  (2) journal-based deleted-dirent recovery is the upgrade path for
  dw-ext's deep scan; jbd2 parsing is a self-contained addition once
  the core engine is in.

## 4. raw_signs.json (signature DB schema)

The carver is configured by a JSON file with one object per extension:

```json
"prproj": {
  "header": { "signs": ["0,1f8b08000000000000,0.96", ...] },
  "tail":   { "signs": [...] },
  "min": 10240, "max": 51200
}
```

Each sign is `<offset>,<hex magic>,<confidence>`; `min`/`max` are file
size bounds. Cross-check against our dw-signatures schema (258 formats, built from
public corpora): our schema already carries every feature above -
header magics with offsets and wildcards, tail magics with
offset-from-end, per-format `min_size`/`max_size`, and per-signature
`weight` (integers 0-1000 in our encoding vs their decimals; same
role). The carver enforces the max-size cap on extents, uses
weight-descending dedup at identical offsets, and the tiered
validated-carve pipeline ranks results. Conclusion: schema parity is
already achieved; the remaining work is periodic coverage auditing
against the public corpora, not schema changes.

## 5. Full engine-module pass (completed)

All 13 engine/orchestration binaries are decompiled and function-indexed
under `re-evidence/` (Ghidra 12.1.4 headless). Findings by module:

### NTFSSearchDLL (1,939 functions)
- Class inventory (RTTI): `CNTFSQuickSearch(Base)` implements
  `IQuickFileSystemSearchController`, `CNTFSSearchController` for the
  complete scan; `CMFTInfo`, `CNTFSFileRecordAttribute`,
  `CNTFSFileRecordDataRunsAttribute`,
  `CNTFSFileRecordFileNameAttribute`, `CResident/CNoneResident
  NTFSDataRunList`, `CNTFSBuildVolumeBitmap`,
  `CStorageDeviceWithCache`.
- References `$LogFile` and `$UsnJrnl`: they parse the NTFS log and
  the USN journal for deep recovery. Ours does not yet (roadmap item,
  same family as our ext4 jbd2 note).

### RAWSearchDLL (1,936 functions) - the carver
- Loads `raw_signs.json` at runtime (config-driven; ours is an
  embedded, build-time DB - deliberate: no runtime file dependency,
  fixed provenance).
- `search_end_min_len` is their per-format minimum end-distance gate;
  functionally our `min_size` bound.
- **134 per-format `*Analyser` classes** (a `BruteForce` namespace of
  structural validators: CADTS, CAPE, CAsf, CDWG, CDXF, CEPS, CFCPEvt,
  CFH11, CHFP, MOBIRawFileInfo, AEP, C7Z, OGG, GIF, ...). Ours ships
  5 (JPEG, PNG, ZIP, MP4, PE). This is our largest quality gap in
  carving; the fix is breadth of validated formats, and it is on the
  engine roadmap. The five we ship are validated against the same
  tiered pipeline, so the gap is count, not rigor.
- Their raw searcher exposes `EUCreateRAWSearcherForCellTest` - a
  factory for unit-cell testing the carver in isolation (a good idea;
  our criterion benches serve the same role).

### FAT/exFAT/HFSPlus SearchDLLs (1,283 / 1,308 / 674 functions)
- Same factory/controller pattern as ext4/NTFS; RTTI confirms the
  split (`CFATQuickSearchWorker`, `CExFatQuickSearch`,
  `CQuickSearchAndBuildBitmap` - the quick pass doubles as
  volume-bitmap builder for the deep pass, exactly like our
  `$Bitmap`/allocation-bitmap capture).
- HFSPlusSearchDLL leaks its original source tree (debug strings):
  `hfspluslib/{hfsplusvolume, hfsplusattribute, hfspluscatalog,
  hfsplusbtree?, hfsplus extents}.cpp` - our dw-hfs module split
  matches one-for-one (volumeheader, attributes?, catalog, btree,
  extents, walker).

### Btrfs / ReFS / Zfs SearchDLLs (458 / 638 / 989 functions)
- All three are real, substantial engines (decompiled and indexed;
  nothing shipped as a stub). Evidence for our roadmap decision: these
  formats are implementable; we will do them one at a time with the
  same fixture-driven validation as ext4, not as pretend support.

### DRW.exe / DRWService.exe (186 / 127 functions)
- Small orchestrators: DRW.exe launches the UI (DRWUI.exe) and the
  service; DRWService.exe hosts the driver IOCTL plumbing
  (IoBuildDeviceIoControlRequest consumers). No recovery logic in
  either - consistent with the driver map in section 2.

## 6. Updated engine implications (our repo)

1. dw-ntfs: `$LogFile`/`$UsnJrnl` deep recovery = next NTFS milestone
   after the shipped $I30 pass.
2. dw-carver: grow the validated-analyser set from 5 toward their 134;
   each analyser lands with its own fixture tests (quality bar: the
   tiered pipeline already enforces structure).
3. dw-ext: jbd2 journal-dirent recovery (documented earlier).
4. Btrfs first among the absent engines (most tractable public spec),
   then ReFS (needs the deepest RE), then ZFS.
