# Utility algorithms and isolation architecture

## Purpose

This artifact extends the recovery engine design with original, safe utility subsystems. It responds to static evidence of separate repair, health, and preview components while deliberately avoiding code or logic extraction from those components.

## A. Drive health collection

### Observed evidence

Disk-health markers show Windows disk enumeration, WMI queries, ATA SMART data/threshold commands, NVMe identify/log retrieval, USB/SATA/SCSI/NVMe transport handling, and vendor-specific attribute display names. The observation supports a layered transport/protocol/normalization architecture.

### Original architecture

```text
Device inventory → protocol reader → raw telemetry record → decoder registry → assessment rules → UI
```

- **Inventory:** documented Windows SetupAPI/Storage APIs enumerate disks and map volumes to physical devices.
- **Protocol reader:** strictly read-only ATA SMART/NVMe/SCSI queries selected from a narrow allowlist. Every command and result length is audited.
- **Raw record:** immutable timestamped bytes plus protocol, device identity, command, and parsing version.
- **Decoder registry:** versioned vendor-family decoders; unknown fields are preserved as raw attributes, never guessed.
- **Assessment:** deterministic rules distinguish critical warnings, media errors, wear, temperature, and “unknown”. Assessments cite raw fields and thresholds.

No health function enables SMART, sends maintenance commands, changes firmware, writes firmware logs, or resets a drive.

## B. Artifact inspection and preview

### Threat model

Recovered files may be malicious, malformed, decompression bombs, polyglots, or designed to exploit parsers. A preview must therefore be optional, out-of-process, no-network, capability-reduced, memory/CPU/size limited, and restartable.

### Contract

```text
Inspect(request {range-or-temp-file, detected_type, limits})
  -> result {thumbnail?, normalized_metadata, diagnostics, output_hash}
```

The UI never embeds or executes a document. Workers get a copied bounded sample unless a test-proven range reader is required. A failure returns diagnostics only.

### Staged support

1. text, images, and safe metadata extraction;
2. PDF page/metadata rendering in a dedicated sandbox;
3. audio/video metadata and thumbnail via a licensed worker;
4. OOXML package manifest inspection;
5. optional document rendering only after sandbox review.

## C. Repair subsystem

### Product rule

Repair operates on an explicit user-selected recovered copy and produces a new output plus JSON repair report. It does not alter the source device, original recovered file, encrypted content without owner-supplied authorization, or protected/DRM content.

### Original repair workflow

1. Verify input identity, type, size, and destination safety.
2. Build a read-only structural model with maximum nesting, object, and decompressed-byte limits.
3. Produce a diagnosis report before any output: intact, truncated, invalid metadata, invalid index, CRC mismatch, unsupported encryption, or unsupported corruption.
4. Perform only format-specific, independently specified transformations.
5. Validate the new output with a separate parser/open pass.
6. Atomically publish the output, record hashes, applied repair actions, warnings, and validator version.

### Initial repair targets

| Target | Safe first operation | Verification |
| --- | --- | --- |
| ZIP | Parse EOCD and central directory; extract internally consistent entries to a new directory. | Per-entry CRC and size. |
| JPEG/PNG | Structural validation and metadata-only rewrite where safe. | Independent decoder open + dimension/scan checks. |
| PDF | Detect and report recoverable xref/object issues; only implement a rewrite after fixture coverage. | Independent parser + object/xref traversal. |
| OOXML | Inspect ZIP package/relationship parts; salvage intact package members. | ZIP CRC and relationship/package validation. |
| Media | Inspect stream/container metadata and remux readable streams to a new container. | Reopen and decode bounded samples. |

## D. Media repair lessons

Static markers in the video utility refer to container/frame parsers, quick versus advanced repair, reference samples, stream extraction, and muxing. The clean-room requirement is to formulate each supported operation as a published container/codec rule:

- MP4/MOV: parse box hierarchy with 64-bit checked ranges; detect missing/corrupt indexes; remux only readable streams.
- AVI: validate RIFF chunks and indexes; rebuild an index only from independently parsed frame boundaries.
- Matroska: parse EBML with recursion/size caps; recover complete clusters only.
- Never invent decodable frames or silently change timestamps; record dropped/invalid regions.

Reference-file-assisted repair is deferred because it is heuristic-heavy and must have clearly defined ownership, privacy, and fidelity tests.

## E. Dependency and license policy

Every parser/codec library needs: source pin, license review, SBOM entry, vulnerability-monitoring path, fuzz-corpus ownership, and sandbox decision. A bundled binary from a reference product is never redistributed or treated as a dependency source.

## F. Utility acceptance gates

- Crash/hang in a worker cannot terminate the core process.
- No worker gets device handles, network access, user credentials, or arbitrary destination access.
- Test fixtures cover malformed, nested, encrypted, truncated, huge, and polyglot inputs.
- Health commands are verified against the static read-only allowlist at test time.
- Repair outputs are reproducible for the same input, options, and dependency versions.
