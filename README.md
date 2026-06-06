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

| Issue | Category | Summary |
|-------|----------|---------|
| 001 | macOS | pyenv fails to compile Python on macOS 26 |
| 002 | Slurm | Job stuck in CG state in Lima VM - cgroup limitation |
| 003 | NWChem | Basis library not found after conda install |
| 004 | conda | numpy conflicts with NWChem in conda environment |
| 005 | Lmod | Module system not persistent across Lima sessions |
| 006 | Lima | matplotlib savefig fails on read-only filesystem |
| 007 | Lima | bash scripts cannot read Lima-mounted macOS filesystem |
| 008 | Lima | submit_and_watch.sh fails with read-only filesystem |
| 009 | Slurm | Node stuck in IDLE+COMPLETING+NOT_RESPONDING state |
| 010 | CI/CD | GitHub Actions shellcheck pipeline - SC2046 and SC1091 |

## Test results

| Script | Status | Notes |
|--------|--------|-------|
| check_disk.sh | OK | Works correctly |
| hpc_health_check.sh | OK | Shows Slurm, disk, memory, load |
| module_check.sh | OK | Detects available/missing modules |
| job_efficiency.sh | OK | sacct disabled in Lima - handled gracefully |
| submit_and_watch.sh | Partial | Blocked by Lima cgroup limitation |

## CI/CD

bash-hpc-toolkit has a GitHub Actions pipeline that runs shellcheck
on every push. Pipeline passes as of 2026-06-06.

## Validated results

| Software | Version | Test | Result |
|----------|---------|------|--------|
| NWChem | 7.3.0 | H2O HF/STO-3G | -74.962946671090 Hartree |
| mpi4py | 4.1.2 | Strong scaling 1-4 ranks | Speedup x2.16 on 2 ranks |
| Apptainer | latest | scientific-python container | numpy 2.4.6, scipy 1.17.1 |
| Lmod | 8.6.19 | module load NWChem/7.3.0 | Working |
| GitHub Actions | - | shellcheck pipeline | Passing |
