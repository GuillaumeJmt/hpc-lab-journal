# HPC Lab Journal

A technical diary of real problems encountered and solved while building
an HPC support environment on Apple M1 (ARM64).

This is not a sanitized tutorial. Every issue is real.

## Why this journal exists

HPC support engineering is mostly about diagnosing unexpected problems.
This journal documents the diagnostic process - what failed, why, and how it was fixed.

## Read the journal

See [JOURNAL.md](JOURNAL.md) for the full log.

## Environment

- Hardware: Apple MacBook M1 (2019) - ARM64
- Host OS: macOS 26 (Tahoe)
- VM: Lima 2.1.1 - Ubuntu 24.04 LTS ARM64

## Issues documented

| Issue | Summary |
|-------|---------|
| 001 | pyenv fails to compile Python on macOS 26 |
| 002 | Slurm job stuck in CG state in Lima VM |
| 003 | NWChem basis library not found after conda install |
| 004 | numpy conflicts with NWChem in conda environment |
| 005 | Lmod not persistent across Lima sessions |
| 006 | matplotlib savefig fails on read-only Lima filesystem |
