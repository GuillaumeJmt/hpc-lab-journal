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
| bash-hpc-toolkit | Defensive bash scripts | 5 scripts with set -euo pipefail |
| slurm-efficiency-analyzer | Job efficiency analyzer | HTML report from sacct data |
| hpc-lab-journal | This journal | Real issues and solutions |

## Validated results

| Software | Version | Test | Result |
|----------|---------|------|--------|
| NWChem | 7.3.0 | H2O HF/STO-3G | -74.962946671090 Hartree |
| mpi4py | 4.1.2 | Scaling methodology | Corrected; results pending validated hardware |
| Slurm | 23.x | Job submission | Functional with workarounds |
| Lmod | 8.6.19 | module load NWChem/7.3.0 | Working |

---

## Issue 007 - bash scripts cannot read Lima-mounted macOS filesystem

**Date:** 2026-06-06

**Error:** Resource deadlock avoided when accessing /Users/guillaumelumin/Documents from Lima

**Root cause:**
Lima mounts the macOS home directory via VirtioFS. Some file operations
(lseek, certain reads) are not supported on this mount type.
bash scripts fail when trying to read files from the macOS filesystem inside Lima.

**Solution:**
Copy files to the Linux home first via limactl cp:
    limactl cp ~/Documents/HPC-ULB/bash-hpc-toolkit/script.sh hpc-node:/home/guillaumelumin.guest/

**Lesson:**
Always work with files in the Linux home inside Lima.
Use limactl cp to transfer files between macOS and Lima.

---

## Issue 008 - submit_and_watch.sh fails with read-only filesystem

**Date:** 2026-06-06

**Error:** tee: submit_watch.log: Read-only file system

**Root cause:**
Script was run from a directory mounted read-only (macOS filesystem via Lima).
tee cannot write log files there.

**Solution:**
Always run scripts from the Linux home directory inside Lima:
    cd ~
    bash ~/submit_and_watch.sh ~/test.sh

**Lesson:**
HPC scripts that write log files must be run from a writable directory.
On real clusters, always run from /home or /scratch, never from NFS read-only mounts.

---

## Issue 009 - Slurm node stuck in IDLE+COMPLETING+NOT_RESPONDING

**Date:** 2026-06-06

**Error:** Node state: IDLE+COMPLETING+NOT_RESPONDING
Jobs stay in PENDING despite node appearing idle.

**Root cause:**
Lima VM has incomplete cgroup support. Jobs that finish abnormally
leave the node in a mixed state that Slurm cannot resolve cleanly.
This is a known Lima/virtualization limitation.

**Workaround:**
    sudo systemctl stop slurmd slurmctld
    sudo rm -rf /var/lib/slurm/slurmctld/hash.*
    sudo systemctl start slurmctld slurmd
    sudo scontrol update NodeName=lima-hpc-node State=idle

**Lesson:**
On real HPC clusters with proper cgroup support, this does not occur.
The Lima environment is sufficient for learning Slurm administration
but has known limitations with job cleanup.

---

## Test results - bash-hpc-toolkit

**Date:** 2026-06-06

All scripts tested in Lima VM Ubuntu 24.04 ARM64.

| Script | Status | Notes |
|--------|--------|-------|
| check_disk.sh | OK | Works correctly, SKIP for non-existent paths |
| hpc_health_check.sh | OK | Shows Slurm status, disk, memory, load |
| module_check.sh | OK | Correctly detects available/missing modules |
| job_efficiency.sh | OK | sacct disabled in Lima - handled gracefully |
| submit_and_watch.sh | Partial | Job submission works, monitoring blocked by Lima cgroup issue |

---

## Issue 010 - GitHub Actions CI pipeline fails on shellcheck warnings

**Date:** 2026-06-06

**Context:** Setting up GitHub Actions CI/CD pipeline to automatically verify
bash script quality with shellcheck on every push.

**Error:**
    In job_efficiency.sh line 23:
        --jobs=$(sacct ...) \
               ^-- SC2046 (warning): Quote this to prevent word splitting.
    In module_check.sh line 16:
    source /etc/profile.d/lmod.sh 2>/dev/null || true
           ^-- SC1091 (info): Not following external file

**Root cause:**
Two real issues detected by shellcheck:
1. SC2046: Unquoted command substitution in --jobs= argument risks word splitting
   if the output contains spaces or special characters
2. SC1091: shellcheck cannot follow external source files not specified as input

**Solution:**
SC2046 - Quote the subshell:
    --jobs="$(sacct -u "$USERNAME" ...)"

SC1091 - Tell shellcheck to ignore this specific source:
    # shellcheck source=/dev/null
    source /etc/profile.d/lmod.sh 2>/dev/null || true

**Lesson:**
shellcheck catches real bugs. SC2046 is a genuine risk - unquoted command
substitution can break scripts in unexpected ways. Always quote subshells.
CI/CD pipelines that run shellcheck on every push prevent these issues
from reaching production.

---

## CI/CD pipeline created

**Date:** 2026-06-06

**Repo:** bash-hpc-toolkit

Created GitHub Actions pipeline (.github/workflows/shellcheck.yml) that:
- Runs shellcheck on all .sh files on every push to main
- Verifies set -euo pipefail is present in all scripts
- Completes in ~11 seconds

Pipeline history:
- Run #1: FAILED - shellcheck found SC2046 and SC1091
- Run #2: PASSED - after fixing both issues
- Run #3: PASSED - confirmed on clean commit

This demonstrates the DevSecOps workflow mentioned in the ULB job posting:
code push -> automated quality check -> fix -> green pipeline.

**Lesson:**
CI/CD pipelines for infrastructure code (bash scripts, Ansible playbooks,
modulefiles) catch issues before they reach production clusters.
The same principle applies to GitLab CI/CD used by the ULB HPC team.

## Issue 011 - Lima VM unreachable: no route to host, boot frozen before network

**Date:** 2026-06-24

**Error:**
    limactl shell hpc-node
    ssh: connect to host 127.0.0.1 port 52000: Connection refused

VM shows STATUS=Running in `limactl list`, but every SSH attempt is refused.
A second VM (lima-test) on the same host worked normally.

**Diagnosis:**
Worked through the three distinct network failure modes.

1. Host side - `Connection refused` on 127.0.0.1:52000
   The port-forward exists but nothing answers behind it.

2. Host agent log (ha.stderr.log):
    tcpproxy: error dialing "192.168.5.15:22": connect tcp 192.168.5.15:22: no route to host
    kex_exchange_identification: read: Connection reset by peer
   `no route to host` (not `refused`) means the guest itself is unreachable on
   the internal network - not just sshd being down.

3. Host agent log (ha.stdout.log):
    {"status":{"vsock":{"type":"failed","reason":"Failed to wait for guest SSH server"}}}
   The guest sshd never came up over vsock.

4. Serial console (serialv.log): 0 bytes, empty even during `limactl start`
   and `limactl --debug start`. The kernel writes nothing -> the guest is not
   booting at all.

**Root cause:**
The VM is "Running" for the hypervisor (the VM process exists) but the guest OS
never boots: the EFI/disk boot state was corrupted, most likely after an abrupt
stop or host sleep while the VM was running. The failure is upstream of the
network and of SSH.

**Workaround attempted:**
    limactl stop --force hpc-node
    limactl start hpc-node
Did not recover - console stayed empty, boot still frozen.

**Resolution:**
Data is safe: the 107 GB `disk` image is intact, and all real work
(slurm.conf, scripts, NWChem setup) is already pushed to GitHub. Rather than
spend time on recovery before the interview, the VM is left aside; a clean
rebuild (~15 min) is the pragmatic option if a sandbox is needed again.

**Lesson:**
- Distinguish the three failure modes: `Connection refused` (host reachable,
  service down -> look at the service), `no route to host` (host/network
  unreachable -> look at the network/VM), `timeout` (filtered -> firewall).
  Here, refused + no route + empty serial console = boot frozen before the
  network came up, NOT an SSH problem.
- A VM reporting "Running" does not mean the guest OS actually booted.
- Take a snapshot of a working VM so a future corruption is a one-command
  rollback instead of a rebuild:
    limactl snapshot create hpc-node --tag ok
    limactl snapshot apply hpc-node --tag ok
  Same "reproducible state" reflex as Infrastructure-as-Code.

---
