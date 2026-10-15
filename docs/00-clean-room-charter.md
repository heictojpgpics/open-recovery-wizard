# Clean-room research charter

## Purpose

This project will implement an original data-recovery system, not a clone. A commercial product may be observed only to understand externally visible behavior and interoperability requirements. Design decisions must be traceable to public filesystem specifications, independently reproduced experiments, or an original engineering rationale.

## Non-negotiable boundaries

- No source, decompiled code, UI resources, icons, localization strings, or other expressive material from a reference product may enter this repository.
- No activation, licensing, update entitlement, or paywall bypassing, patching, emulation, credential extraction, or tampering.
- No execution of an untrusted installer on a production host. Dynamic observation, when needed, must use a disposable Windows VM with a synthetic test corpus and snapshot rollback.
- No writes to a recovery source device. Read-only handles are mandatory; writes are restricted to an explicit export destination that is rejected if it aliases the source disk.
- All claims are labelled as **observed**, **inferred**, or **planned**. Inference is never treated as recovered source logic.

## Evidence handling

Provided installers and extracted payloads are retained only in a local, ignored evidence directory. Their identities are recorded as SHA-256 hashes in the study document. Public commits contain only independently written notes, aggregate metadata, and original work.

## Decision standard

The implementation must be defensible without access to any reference binaries. Every feature needs an independently written specification, fixture, expected result, and reproducible test before implementation is accepted.
