# Validation and safety plan

## Test corpus

Use only synthetic, self-created images for automated tests. Build fixture families for FAT32, exFAT, NTFS, EXT4, MBR, GPT, sparse files, deleted files, intentionally corrupt metadata, and unallocated raw files. Each fixture has a manifest with source creation recipe, known files, deletion method, expected results, and SHA-256 digest.

## Correctness matrix

| Test | Acceptance criterion |
| --- | --- |
| Source immutability | Pre/post block hashes match exactly after every scan, preview, cancellation, and export test. |
| Enumeration | Partition tables and filesystem detection match fixture manifests; malformed inputs produce typed errors. |
| Recovery | Expected recoverable fixtures export with matching size and SHA-256; known overwritten fixtures are not overstated. |
| Carving | Header-only false positives are rejected by validators; footer/size ambiguity is recorded in confidence/corruption state. |
| Resume | Cancel/resume yields the same result set and no duplicate entries as uninterrupted scan. |
| Destination safety | Same physical disk and source path aliases are rejected before export. |
| Resource controls | Peak memory, cancellation latency, and progress cadence are measured on CI/nightly fixture tiers. |
| Robustness | Fuzzers and corpus mutations never crash, hang, write the source, or exceed configured memory limits. |

## Windows test environment

Use ephemeral Windows runners or a disposable VM for build, installer, UI smoke, and image-file integration tests. Tests must not attach real disks or personal data. Keep a binary-analysis VM isolated from credentials and production networks. Capture test logs and manifests, but never upload vendor binaries or device contents as build artifacts.

## Release artifacts

The intended Windows release channels are:

- portable ZIP for diagnostic and sandbox use;
- signed installer for normal distribution;
- MSIX only after device-access and packaging behavior have been verified.

Each release must publish checksums, SBOM, license notices, test summary, and reproducible build instructions. Code signing and MSIX are not prerequisites for the first engine milestones.
