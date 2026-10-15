# Engineering program: build artifacts and verification ladder

## Repository artifacts to create before feature code

```text
docs/specs/                 Independently written format and behavior specs
fixtures/manifests/         Synthetic source-image recipes and expected results
fixtures/generators/        Deterministic fixture builders
crates/recovery-domain/     Types and errors only
crates/recovery-blockio/    Read-only source implementation
crates/recovery-partition/  MBR/GPT discovery
crates/recovery-session/    Checkpoints and result store
crates/recovery-fs-*/       Filesystem engines
crates/recovery-carve/      Original signature engine
crates/recovery-health/     Read-only drive telemetry
workers/                    Preview and repair worker boundaries
apps/desktop/               Tauri 2 + React application
fuzz/                       Parser-specific fuzz targets
bench/                      Reproducible workloads and metrics
```

## Work packages

| Package | Deliverable | Exit criterion |
| --- | --- | --- |
| P0 | Threat model, source immutability harness, fixture manifest schema. | Scan cancellation cannot mutate any test source. |
| P1 | Read-only image I/O plus MBR/GPT. | Known-good and malformed partition fixtures pass. |
| P2 | Result store, export safety, basic desktop job UI. | Export rejects source-disk destinations and is atomic. |
| P3 | FAT32/exFAT metadata recovery. | Deleted/non-deleted fixture matrix with hash-matched exports. |
| P4 | Raw carver and validator framework. | Header-only false positives are rejected by tests. |
| P5 | NTFS metadata recovery. | Fixups/runlists/bitmap cases covered by property and fixture tests. |
| P6 | EXT4 recovery and durable resume. | Interrupted/resumed result set matches uninterrupted scan. |
| P7 | Health telemetry and preview sandbox. | Workers cannot access source-device handle or network. |
| P8 | Repair utilities. | Every repair action produces a validated copy and report. |
| P9 | Physical-device hardening, packaging, SBOM, reproducible release. | Windows integration and security review sign-off. |

## CI policy

Pull requests run formatting, lint, unit/property tests, fixture integration tests, and small fuzz regression corpora. Nightly runs run extended fuzzing, large fixture scans, performance budgets, dependency audit, SBOM generation, and Windows smoke tests. Release builds require an artifact checksum manifest and reproducible build comparison where toolchains permit.

## Definition of done

A feature is done only if its independent specification, fixture creation recipe, expected results, test reports, telemetry/privacy review, and user-facing limitation text are committed with it. “Recovered successfully” is never sufficient without integrity evidence.
