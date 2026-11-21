# Karkain Process Manager

A planned CLI/TUI-first tool for deterministic, scriptable process inspection and diffable process-graph export, built exclusively in Karkain v1.2.0.

## Status

| Area | Status |
| --- | --- |
| Research | COMPLETE |
| Product design | COMPLETE |
| Repository creation | COMPLETE |
| Implementation | NOT STARTED |
| Compile/execute verification of the application | NOT STARTED (toolchain itself VERIFIED on Karkain v1.2.0) |

## What it does

Karkain Process Manager is intended to inspect the processes running on a host and emit structured, reproducible snapshots: a process list, a parent/child relationship graph rendered as text, and per-process attributes when a data source is available. Every record is tagged with its source and verification status. Karkain v1.2.0 has no portable process-enumeration API: the only host-interaction primitive is a `system()` shell-out whose stdout capture is UNVERIFIED. The design is therefore probe-gated: a field is emitted only if its probe passes; otherwise the field is recorded as `unverified/unavailable`. Handle and DLL forensics, wait-chain analysis and any GUI process tree are out of scope because the required OS APIs and GUI toolkit do not exist in the language. This product is per-process; host-level CPU/memory/storage monitoring is the separate Karkain System Monitor product.

## Who it serves

[RI] Developers, power users and analysts who need machine-readable process facts for scripts, tests and reproducible fixtures; terminal-first operators who cannot use a GUI tool; and teams mixing Windows and Unix-like hosts who want one output format. It is not for users needing interactive handle/DLL forensics or a live GUI tree (BLOCKED).

## Planned features

- Inspection (planned): probe-gated process enumeration adapter over documented shell-outs (planned); per-record source and status labels (planned); configurable field set (planned).
- Process graph (planned): parent/child relationship extraction where a source provides it (planned); deterministic text-tree rendering (planned); cycle/duplicate handling rules (planned).
- Export (planned): structured snapshot output via the PARTIAL `json` module (planned); diffable text snapshot (planned); fixture export for tests (planned).
- Actions (planned): dry-run command planning for start/stop-style operations (planned); explicit confirmation and audit logging (planned). The default is read-only.
- Diagnostics (planned): `probe` subcommand reporting adapter readiness (planned); self-test for deterministic output (planned).
- Platform (planned): Windows x86-64 first (planned); Linux/macOS only after execution verification (planned).

## Explicit non-goals

- No GUI process tree, windows or controls (no native GUI toolkit; BLOCKED).
- No handle, DLL, symbol or kernel-memory forensics (no process APIs; BLOCKED).
- No wait-chain/stack analysis in the first releases.
- No native process-enumeration claim while output capture remains UNVERIFIED.
- No host-level resource metrics (that is Karkain System Monitor).
- No destructive-by-default actions and no fabricated process records.
- No modification of the Karkain language or its stdlib.

## Karkain foundation

Exclusive foundation: Karkain Language v1.2.0. All application source will be Karkain. No other implementation language. No compiler modifications.

## Platform support status

Windows x86-64 toolchain VERIFIED (C-free native target with the Go front-end). Linux cross-target output PRESENT but execution NOT VERIFIED on this host. macOS NOT VERIFIED. Process enumeration itself is NOT VERIFIED on any platform because no probe has been run.

## Build and run status

No application binary exists yet. The intended generic commands are `karkain build src/main.kark -o kprocess` and `karkain run src/main.kark`; these become active when implementation starts. The verified toolchain baseline on this machine uses `KARKAIN_ENGINE=go` with `--target native-x86_64-windows`, which compiles and runs a real executable. Application-level compile/run verification is still NOT STARTED.

## Known blockers

- OS process enumeration APIs: BLOCKED/PARTIAL - only `system()` shell-outs; stdout capture UNVERIFIED.
- Handle/DLL inspection: BLOCKED - no process/handle builtins.
- Native GUI toolkit: BLOCKED - CLI/TUI only.
- `json` module: PARTIAL - no full DOM, streaming or array fidelity guarantee.
- Default `kcc` engine: BLOCKED in this environment; use the Go engine + C-free native target.
- Linux/macOS execution: NOT VERIFIED.

## Repository

Source: https://github.com/ajit-ai/Karkain-Process-Manager
Issues: https://github.com/ajit-ai/Karkain-Process-Manager/issues
License: Apache-2.0.
