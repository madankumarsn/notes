---
topic: NVIDIA driver/library version mismatch breaking GPU Docker containers after a system update
status: draft
layer: debug-log
---
<!-- Generative AI was used in the Creation/Modification of this file -->

# NVIDIA driver/library mismatch breaks GPU Docker container after a system update

**One-line idea:**
`nvidia-container-cli: initialization error: nvml error: driver/library version mismatch` means the currently-loaded NVIDIA kernel module and the installed userspace driver libraries disagree — almost always because an in-place driver package upgrade landed without the kernel module actually being reloaded (i.e. no real reboot since the update), and a reboot only fixes it if it actually boots the newly-installed kernel/initramfs.

**Why it exists:**
A recurring, generically-named error whose real cause (kernel module vs. userspace library skew) isn't obvious from the message, and whose "just reboot" fix can silently fail to apply if the reboot doesn't actually pick up the new kernel.

**Math / mechanics:**
NVIDIA's driver stack has two halves that must agree in version: the kernel module (`.ko`) actually loaded into the running kernel, and the userspace libraries/NVML that programs like `nvidia-smi` and the container runtime (`nvidia-container-cli`) link against. An in-place package upgrade (`apt upgrade`) replaces the files on disk — new libraries, new DKMS module source — but the *old* kernel module stays loaded in memory until something actually triggers a reload, which in practice means booting into a kernel/initramfs built after the upgrade. Until that happens, the on-disk (new) and in-memory (old) versions disagree, which is exactly what the mismatch error reports.

**Code:**
```text
$ docker/base/run-gpu.sh
docker: Error response from daemon: ... OCI runtime create failed ...
nvidia-container-cli: initialization error: nvml error: driver/library version mismatch

$ nvidia-smi
Failed to initialize NVML: Driver/library version mismatch
NVML library version: 580.173
```

```bash
# Diagnose: is the loaded kernel module actually older than the installed libraries?
cat /proc/driver/nvidia/version
dpkg -l | grep nvidia-driver        # installed userspace package version
modinfo nvidia | grep ^version      # version the currently loaded .ko reports

# Did the "reboot" actually reboot into the new kernel/initramfs?
uptime -s                           # boot time
uname -r                            # currently running kernel
ls -la /boot/initrd.img-$(uname -r) # check initramfs timestamp vs package install time
grep -i nvidia /var/log/apt/history.log | tail    # when the driver packages actually changed

sudo reboot
# after it comes back:
uname -r
nvidia-smi
docker/base/run-gpu.sh
```

**Gotchas:**
- The error itself gives no hint that it's a reboot/module issue — it reads like a Docker or GPU-container-toolkit misconfiguration, but the fix is almost always "reload the kernel module to match the installed libraries," not a Docker setting.
- A reboot is **not guaranteed to fix it** if the reboot happened *before* the driver package finished updating, or if the bootloader's default entry still points at an older kernel — check `uptime -s` against the package install timestamp, and `uname -r` against the installed kernel version, before concluding "I already rebooted and it's still broken."
- Installing all NVIDIA meta-packages at once (driver + utils + dev, etc.) in a single `apt` transaction risks a partially-consistent set if the transaction is interrupted — worth checking `dpkg -l | grep nvidia` for version consistency across packages if the mismatch persists after a clean reboot.

**Related:** [[_debug-log-hub]]

**Still unclear:**
Whether the second reboot (into the correct kernel) actually resolved this specific instance wasn't confirmed in the source material — verify the diagnostic commands above actually point at the root cause before treating this as a closed case, and re-check if GPU containers break again after a future system update.
