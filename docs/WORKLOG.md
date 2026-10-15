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

## 2026-10-15 — utility subsystem extension

- Expanded static analysis to the supplied file-repair, disk-health, media-viewing, Office-viewing, and PDF-viewing bundles.
- Recorded evidence for separate repair workers, Qt/QML health telemetry, vendor/transport-aware SMART and NVMe handling, FFmpeg-based media components, and dedicated preview processes.
- Extended the original architecture with read-only health collection and isolated preview/repair brokers. Added strict no-mutation, no-encryption-bypass, licensing, provenance, and worker-isolation rules.
- Created the public `heictojpgpics/open-recovery-wizard` repository and pushed the clean-room documentation baseline.

## 2026-10-15 — algorithm-family reconstruction

- Performed additional static interface and API analysis of recovery modules, filesystem scanner markers, raw-signature configuration, repair workers, and health/viewer components.
- Converted the evidence into original scanner, carver, confidence, checkpoint, health, preview, repair, and build-program specifications.
- Preserved the clean-room rule: no vendor code, signature table, visual assets, protected logic, or activation-related behavior is copied into this project.
