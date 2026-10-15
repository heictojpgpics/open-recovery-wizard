# Recovery algorithm reconstruction: evidence to original design

## Status and method

This is a clean-room engineering artifact, not recovered source code. It combines: static module/interface evidence from the supplied local payload, public on-disk format knowledge to be independently cited during implementation, and original design decisions. Claims are marked **observed**, **inferred**, or **planned**.

## 1. Observed scanner boundary

**Observed.** The supplied filesystem modules use a common vocabulary: filesystem search controller, quick-search controller, complete-search controller, searched partition information, file node, file properties, and data-run list. The raw-search component exports factories/analyser-map functions but does not itself import the normal Windows file-read APIs used by several filesystem modules. This supports the inference that a storage abstraction is passed into scanner plug-ins.

**Observed.** Named scanner factories exist for NTFS, FAT, exFAT, EXT4, Btrfs, ReFS, ZFS, and HFS+. All but the Btrfs spelling expose a quick and a standard/complete factory. Markers identify NTFS MFT/data runs/volume bitmap; FAT 12/16/32 tables; exFAT file sets/bitmap; EXT4 journal/inodes; ReFS metadata/checkpoint trees; ZFS vdev/uberblock/ZPL structures; and HFS+ catalog/extents/attributes/journal B-trees.

**Inference.** The product likely runs a metadata-oriented traversal before or alongside a broader scan. This does not establish its ordering or heuristics.

## 2. Original scanner contract

The project will use an explicit Rust contract, not a binary-compatible interface:

```text
ScanEngine
  detect(source, partition) -> Detection
  quick_scan(ctx, sink) -> ScanSummary
  deep_scan(ctx, sink) -> ScanSummary
  resume(ctx, checkpoint, sink) -> ScanSummary

FindingSink
  publish(batch: Vec<Finding>) -> backpressured result
  checkpoint(state: ScannerCheckpoint) -> atomic result

BlockSource
  identity() -> SourceIdentity
  geometry() -> Geometry
  read_exact(range) -> bounded bytes
```

`BlockSource` owns all device access. Filesystem and carver modules receive only this read-only capability. Every range is checked against source length before allocation or I/O.

## 3. Common recovery pipeline

### Phase A — source identity and read safety

1. Open images directly; open physical disks only with documented read-only Windows flags.
2. Record source type, length, logical/physical sector size, removable state, and a fingerprint from fixed sampled ranges.
3. Reject a source if its identity changes between start, resume, preview, or export.
4. Make source writes impossible at the type/API boundary; no `write` method exists in production block-source traits.

### Phase B — partition and volume discovery

1. Parse MBR and GPT using checked reads.
2. Validate headers, table bounds, CRCs where applicable, overlap, and alignment.
3. Probe each candidate volume by independent filesystem signatures and plausibility rules.
4. Preserve candidates with provenance and confidence rather than selecting one silently.

### Phase C — metadata recovery

For a recognized filesystem: parse only required metadata first, validate all parent/child references, emit directory/file nodes incrementally, and represent physical content as canonical extents. A file is not marked exportable until its extents fit within the source and do not violate filesystem constraints.

### Phase D — unallocated/deep recovery

The deep phase receives a precise scan plan: unallocated extents when a trustworthy allocation map exists; otherwise sector-aligned source ranges. Chunk boundaries overlap by the maximum signature length minus one. Findings are deduplicated by `(source, start offset, parser/version)` and then reconciled with metadata results.

### Phase E — validation, preview, export

Validate structural invariants before preview. Preview uses a separate worker. Export reads the original extents again, writes a temporary target, records byte count and hash, flushes it, and atomically publishes it. The destination must not map to the source physical disk.

## 4. Filesystem implementation plan

| Filesystem | Evidence-led learning target | Original algorithm plan | First release |
| --- | --- | --- | --- |
| FAT12/16/32 | FAT tables, root/folder analysis, allocation concepts. | Parse BPB; validate FAT copies; traverse directories; follow cluster chains with loop/size bounds; recover deleted directory entries only when chain plausibility remains. | FAT32 first, then FAT12/16. |
| exFAT | File-set, bitmap, allocation, and partition markers. | Parse boot region/checksum; validate allocation bitmap and file-entry sets; reconstruct contiguous/no-FAT-chain streams only with validated bounds. | Yes. |
| NTFS | MFT, resident/non-resident data runs, volume bitmap, filename attributes. | Parse boot sector/MFT record fixups; validate attribute headers; decode bounded signed runlists; cross-check bitmap and record state; preserve alternate streams as separate manifest entries. | Yes, after FAT/exFAT. |
| EXT4 | volume, journal traversal, inode/data-run markers. | Validate superblock/group descriptors; traverse inode tables/directories; decode extents; use journal only as an explicitly labelled supplemental evidence source. | Yes, after NTFS. |
| ReFS | metadata block, superblock, checkpoint, tree/node markers. | Research-only until independently documented fixture coverage exists; parse read-only structures with full checksum/reference validation. | Deferred. |
| Btrfs/ZFS/HFS+ | module markers show format-specific tree/object concepts. | Research-only; model copy-on-write generation/transaction consistency before file recovery. | Deferred. |

No parser will infer a parent path, deletion state, or exact file length if metadata conflicts. The result instead exposes the ambiguity.

## 5. Raw-carving findings and original carver

### Observed signature semantics

The supplied JSON contains 139 file-type definitions, all with header signatures; 46 also define tail signatures. It contains 199 header patterns and 62 tail patterns. Definitions carry size bounds and some patterns start at non-zero offsets. Forty-four definitions allow multiple headers and eleven multiple tails. This strongly supports declarative magic-number matching with offset, alternative-pattern, tail, and size controls. It does not prove match scoring or reconstruction rules.

### Planned signature schema

```json
{
  "id": "org.example.jpeg",
  "version": 1,
  "headers": [{"offset": 0, "bytes": "ffd8ff"}],
  "length": {"min": 4, "max": 268435456},
  "validator": "jpeg",
  "footer": {"bytes": "ffd9", "strategy": "validated_first"}
}
```

The project creates its own signatures from published format specifications and its own fixtures. It does not copy the supplied table into the repository.

### Planned carver algorithm

1. Compile header patterns into a multi-pattern matcher; retain required offsets and validator ID with each match.
2. Stream scan fixed-size chunks with exact overlap; never concatenate whole-disk buffers.
3. Convert a header match to a candidate only if its relative offset/range is valid.
4. Select an end using a format parser, declared length, validated footer, or bounded fallback, in that priority order.
5. Run a format-specific validator under a CPU/byte budget. Validate internal length fields, checksums/CRCs where specified, nested object bounds, and terminal markers.
6. Score a finding from explicit factors: header strength, parser validity, footer validity, contiguous extent, overlap conflict, and truncation. The factor list is stored with the result.
7. Resolve overlaps deterministically: a stronger validated candidate wins only when it contains the other candidate's range or format policy permits nesting; otherwise retain both with conflict state.
8. Never advertise fragmented reconstruction as a normal recovery. It is a separate research mode with a clear experimental label.

## 6. Checkpoint and concurrency design

Each job owns one coordinator, a bounded I/O queue, a bounded CPU-validation queue, and one transactional result writer. Readers can prefetch only a configured byte budget. Checkpoints include: source identity, partition range, scanner build ID, signature-set version, completed ranges, emitted-finding high-water mark, and hash of critical scan options. A resume is rejected on any mismatch.

## 7. Confidence model

`FindingConfidence` is not a percentage prediction. It is an explainable set of evidence:

- `metadata_validity`: passed/failed/partial validation;
- `allocation_state`: allocated/deleted/unknown;
- `extent_completeness`: exact/truncated/non-contiguous/unknown;
- `container_validation`: none/header-only/structural/decoded;
- `conflicts`: overlap, duplicate, or metadata disagreement;
- `limitations`: known unsupported feature, encrypted/compressed extent, fragment risk.

The UI may derive a simple badge, but the full evidence is always available.

## 8. Acceptance gates before implementation

1. A public specification/reference and a synthetic fixture exist for every parsed structure.
2. Unit, property, and corruption tests cover every offset/length calculation.
3. Source-image hashing proves read-only behavior during scan, preview, cancellation, and export.
4. Fuzzing runs with sanitizer-enabled test builds where supported.
5. Performance reports include throughput, peak RSS, allocation count, cancellation latency, and result-index growth.
