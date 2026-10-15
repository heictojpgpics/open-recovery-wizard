# Worklog

## 2026-10-15 — clean-room research baseline

- Downloaded and integrity-tested two supplied product payload archives; merged their complementary contents only in a local ignored evidence area.
- Downloaded separate Windows installer and macOS DMG. Identified the installer statically as NSIS v3.03; did not execute it.
- Performed static PE/configuration inspection of the primary UI, service, filesystem scan, raw-carving, and partition modules. The study found a native 64-bit Qt deployment with filesystem-specific scanner modules, a raw-signature configuration, session artifacts, preview-related components, and recovery workflow vocabulary.
- Recorded evidence/inference boundaries and excluded activation inspection, any circumvention, vendor code/assets/strings, and dynamic execution from this baseline.
- Established original Rust/Tauri architecture, source-immutability rule, phased filesystem scope, and fixture-driven validation plan.

## Next work items

1. Create the Rust workspace and typed domain model.
2. Add fixture generator and source-immutability test harness.
3. Implement read-only image source and MBR/GPT discovery.
4. Implement safe export guardrails before any recovery parser.
5. Add FAT32/exFAT scanner with property and integration tests.
6. Create a controlled, disposable Windows observation plan for externally visible workflow comparison.
