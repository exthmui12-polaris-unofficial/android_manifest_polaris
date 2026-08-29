Build Environment

Source:

/root/android/exthmui12

Host:

OrangeVPS
16 vCPU
32 GB RAM
320 GB disk
Debian 13

The VPS has sustained CPU usage restrictions.

CPU Restriction

Do not control sustained CPU usage primarily through Ninja/Make -j.

The entire Android build process tree should instead be restricted to 12 logical CPUs.

On this 16-vCPU VPS, use CPUs 0-11.

A convenient interactive method is:

cd /root/android/exthmui12
taskset -c 0-11 bash

Inside that restricted shell:

source build/envsetup.sh
lunch exthm_polaris-userdebug

Verify the effective CPU set:

taskset -pc $$
nproc

nproc should normally report:

12

Then perform a full build normally:

mka bacon

All child processes inherit the restricted CPU affinity.

Exit the restricted shell after the build:

exit

For automated/non-interactive execution, a full build may instead be run as:

taskset -c 0-11 bash -lc '
cd /root/android/exthmui12 &&
source build/envsetup.sh &&
lunch exthm_polaris-userdebug &&
mka bacon
'

Do not run mka bacon outside the restricted CPU context for a full build.

Why Not -j

Commands such as:

mka bacon -j8

only limit build job concurrency.

They do not reliably limit actual CPU utilization because different build tools and build stages can have their own parallelism.

CPU affinity provides the actual resource boundary.

Incremental Builds

Prefer incremental builds whenever possible.

Use the smallest valid target that can verify the source change.

Do not perform a complete bacon build when a module or image rebuild is sufficient.

For CPU-intensive incremental builds, apply the same 12-CPU affinity rule.

Cleaning

Do not routinely run:

rm -rf out
make clean
mka clean

Preserve existing build output and let Ninja reuse it.

Only clean specific generated state when the type of source/configuration change actually requires it.

For genuine Soong-regeneration issues, targeted cleanup may include:

rm -rf out/soong out/.module_paths
rm -f out/build-exthm_polaris.ninja
rm -f out/build-exthm_polaris-cleanspec.ninja

Do not perform this automatically for normal source changes.

Monitoring

Useful commands:

htop
top
df -h
du -sh out 2>/dev/null

After CPU affinity is applied, seeing the allowed 12 CPUs fully utilized during compilation is normal.

The objective is to prevent the build from consuming the remaining 4 VPS CPUs.

Also monitor:

CPU steal
I/O wait
RAM
swap
free disk space
Build Errors

Always locate the first meaningful fatal error.

Root-user nsjail messages such as:

Process will be UID/EUID=0 in the global user namespace
Process will be GID/EGID=0 in the global user namespace

are warnings unless nsjail itself actually terminates the build.

Final messages such as ninja failed or ckati failed are usually consequences rather than root causes.