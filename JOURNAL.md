# HPC Lab Journal

Personal log of technical problems encountered and solved while building
a realistic HPC support environment on Apple M1 (ARM64).

This journal documents real issues - not a sanitized tutorial.

---

## Environment

- Hardware: Apple MacBook M1 (2019) - ARM64
- Host OS: macOS 26 (Tahoe)
- VM: Lima 2.1.1 - Ubuntu 24.04 LTS ARM64
- Goal: reproduce an HPC cluster environment for learning and portfolio

---

## Issue 001 - pyenv fails to compile Python on macOS 26

**Date:** 2026-05-29

**Error:** BUILD FAILED (OS X 26.2 using python-build 2.6.31)

**Root cause:**
macOS 26 (Tahoe) is too recent for pyenv 2.6.31. Build scripts do not
yet support the new SDK paths introduced in macOS 26.

**Solution:**
Use Apple system Python 3.10 directly. Acceptable for our use case.

**Lesson:**
On cutting-edge OS versions, package managers may lag behind.
Always check compatibility before spending time debugging build failures.

---

## Issue 002 - Slurm job stuck in CG (Completing) state

**Date:** 2026-05-30

**Error:** Job stays in CG state indefinitely, output file never created.

**Root cause:**
Lima VM has incomplete cgroup support. Slurm uses cgroups via
ProctrackType to track and kill job processes. Cleanup fails silently
in the virtualized environment.

**Solution:**
    sudo systemctl stop slurmd slurmctld
    sudo rm -rf /var/lib/slurm/slurmctld/hash.*
    sudo systemctl start slurmctld slurmd
    sudo scontrol update NodeName=lima-hpc-node State=idle

**Lesson:**
Slurm cgroup integration requires full kernel cgroup support.
In virtualized environments, proctrack plugins may behave unexpectedly.
On real HPC clusters this issue does not occur.
Document the limitation rather than hiding it.

---

## Issue 003 - NWChem basis library not found

**Date:** 2026-05-31

**Error:** bas_tag_lib: failed opening basis file

**Root cause:**
NWChem installed via conda-forge stores basis libraries at a different
path than the one hardcoded at compile time.

**Diagnosis:**
    find ~/miniconda3 -name "sto-3g" -type f
    # Found: .../miniconda3/share/nwchem/libraries.bse/

**Solution:**
    export NWCHEM_BASIS_LIBRARY=~/miniconda3/share/nwchem/libraries.bse/

Made permanent in the Lmod modulefile via setenv().

**Lesson:**
conda-forge builds of scientific software often have hardcoded paths
from the build environment. The fix belongs in the modulefile,
not in user scripts.

---

## Issue 004 - numpy conflict with NWChem conda environment

**Date:** 2026-05-31

**Error:** LibMambaUnsatisfiableError - plumed/BLAS conflict

**Root cause:**
NWChem depends on plumed which requires a specific BLAS implementation
that conflicts with numpy requirements in the same conda environment.

**Solution:**
    pip install mpi4py numpy pandas matplotlib scipy

Install via pip instead of conda to bypass the dependency solver.

**Lesson:**
conda environments with complex scientific packages have tight
dependency constraints. pip can coexist with conda for packages
that do not conflict at the binary level.

---

## Issue 005 - Lmod not persistent across Lima sessions

**Date:** 2026-05-31

**Error:** module: command not found after reopening Lima shell.

**Root cause:**
Lmod initialisation script not sourced automatically in non-login shells.

**Solution:**
    source /etc/profile.d/lmod.sh
    module use ~/modulefiles

**Lesson:**
On real clusters, lmod.sh is sourced automatically for all login shells.
This is why HPC documentation always tells users to add module commands
to their ~/.bashrc.

---

## Issue 006 - matplotlib savefig fails with read-only filesystem

**Date:** 2026-05-31

**Error:** OSError: [Errno 30] Read-only file system: '/Users/guillaumelumin/Documents/mpi_scaling.png'

**Root cause:**
Lima VM mounts the macOS home directory as read-only by default.
Files must be saved inside the Linux home (/home/guillaumelumin.guest/)
and then copied to macOS via limactl cp.

**Solution:**
    # Save inside Linux home
    plt.savefig('/home/guillaumelumin.guest/mpi_scaling.png')
    # Copy to macOS
    exit
    limactl cp hpc-node:/home/guillaumelumin.guest/mpi_scaling.png ~/Documents/

**Lesson:**
Always save generated files to the Linux home inside Lima.
Use limactl cp to transfer files to macOS when needed.

---

## Repos created

| Repo | Description | Key files |
|------|-------------|-----------|
| hpc-job-templates | Annotated Slurm job scripts | 7 scripts covering serial/OMP/MPI/array/GPU |
| slurm-admin-cheatsheet | Admin command reference | Commands, parameters, troubleshooting |
| lmod-demo-environment | Lmod modulefile demo | NWChem 7.3.0 modulefile, validated calculation |
| mpi-scaling-benchmark | MPI strong scaling test | mpi4py benchmark, scaling plot |
| bash-hpc-toolkit | Production bash scripts | 5 scripts with set -euo pipefail |
| slurm-efficiency-analyzer | Job efficiency analyzer | HTML report from sacct data |
| hpc-lab-journal | This journal | Real issues and solutions |

## Validated results

| Software | Version | Test | Result |
|----------|---------|------|--------|
| NWChem | 7.3.0 | H2O HF/STO-3G | -74.962946671090 Hartree |
| mpi4py | 4.1.2 | Strong scaling 1-4 ranks | Speedup x2.16 on 2 ranks |
| Slurm | 23.x | Job submission | Functional with workarounds |
| Lmod | 8.6.19 | module load NWChem/7.3.0 | Working |
