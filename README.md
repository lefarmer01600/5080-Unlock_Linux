# HP OMEN Transcend 14 8C58 RTX 4060 — Linux GPU Power Unlock

A hardware-specific Linux GPU power unlock for the **HP OMEN Transcend 14-fb0xxx with system board 8C58 and GeForce RTX 4060 Laptop GPU**.

This repository is a fork of the `original-gpu-unlock-only` branch of the upstream `5080-Unlock_Linux` project, adapted for the OMEN Transcend 14 8C58 platform.

The original project provides the `omen_wmi_boost` kernel module. This fork adds the 8C58-specific changes required for the RTX 4060 to use the laptop's full **65 W GPU power budget** under Linux.

This does **not** overclock the GPU. It enables firmware-controlled performance and power-budget settings that are already supported by the laptop.

## Important

This fork is intended for:

* HP OMEN Transcend Gaming Laptop 14-fb0xxx
* System board `8C58`
* NVIDIA GeForce RTX 4060 Laptop GPU
* Fedora Linux
* KDE Plasma / Wayland

Do **not** use this fork blindly on another HP laptop.

Different OMEN/Victus models can use different WMI commands, firmware layouts, power limits, and thermal-profile values.

The module is an out-of-tree kernel module and therefore taints the kernel when loaded. This is expected.

---

# What this fixes

On the OMEN Transcend 14 8C58, Linux can leave the RTX 4060 in a reduced GPU power state.

The laptop's RTX 4060 configuration is capable of approximately:

```text
50 W base power
+
up to 15 W Dynamic Boost
=
65 W maximum GPU power budget
```

On the affected Linux configuration, the GPU may initially remain around:

```text
50 W
```

even though the NVIDIA driver reports:

```text
Max Power Limit: 65 W
```

This fork enables the HP firmware performance path required for Dynamic Boost to use the additional budget.

Under a sufficiently heavy game workload, the GPU should be able to reach:

```text
Current Power Limit: 65.00 W
```

Actual power consumption will vary with GPU load, CPU load, temperature, and Dynamic Boost.

---

# How the 8C58 fix works

The fork uses HP's WMI firmware interface through the `omen_wmi_boost` kernel module.

The original module enables the HP GPU performance flags:

```text
CTGP
PPAB
```

and applies the OMEN performance profile.

For the 8C58 platform, the correct OMEN performance-profile value is:

```text
0x31
```

rather than the default:

```text
0x01
```

This fork therefore uses:

```text
thermal_profile=0x31
```

The fork also sends HP's Smart Performance Gain WMI command and sets its concurrent CPU/GPU power-budget value to:

```text
45
```

The module logs this operation as:

```text
Smart Performance Gain set to 45
```

These two 8C58-specific changes are the important differences from the original GPU-unlock-only configuration.

---

# Hardware verification

Before installing, verify that the laptop is the expected platform.

Run:

```bash
sudo dmidecode -s system-product-name
sudo dmidecode -s baseboard-product-name
sudo dmidecode -s bios-version
```

Expected on the tested system:

```text
OMEN Transcend Gaming Laptop 14-fb0xxx
8C58
F.10
```

Also check the GPUs:

```bash
lspci -k | grep -EA3 'VGA|3D|Display'
```

Expected:

```text
Intel Corporation Meteor Lake-P [Intel Arc Graphics]
    Kernel driver in use: i915

NVIDIA Corporation AD107M [GeForce RTX 4060 Max-Q / Mobile]
    Kernel driver in use: nvidia
```

---

# Requirements

## Fedora packages

Install the build tools and matching kernel headers:

```bash
sudo dnf install gcc make kernel-devel-$(uname -r)
```

Verify the kernel build directory exists:

```bash
ls -ld /lib/modules/$(uname -r)/build
```

The directory must exist.

## NVIDIA driver

The NVIDIA driver must already be installed and working.

Test:

```bash
nvidia-smi
```

The RTX 4060 should be detected.

## NVIDIA Dynamic Boost

Check:

```bash
cat /proc/driver/nvidia/gpus/0000:01:00.0/power
```

The expected result includes:

```text
Notebook Dynamic Boost: Supported
```

## nvidia-powerd

Dynamic Boost requires the NVIDIA power daemon on supported notebooks.

Check:

```bash
systemctl status nvidia-powerd --no-pager
```

It should show:

```text
Active: active (running)
```

Check that it is enabled:

```bash
systemctl is-enabled nvidia-powerd
```

Expected:

```text
enabled
```

Enable it if necessary:

```bash
sudo systemctl enable --now nvidia-powerd.service
```

---

# Secure Boot

Check:

```bash
mokutil --sb-state
```

Unsigned out-of-tree kernel modules will not load when Secure Boot enforcement is active unless the module is signed and its signing key is trusted.

The simplest option is to disable Secure Boot.

Advanced users can instead configure module signing/MOK enrollment.

---

# Clone the repository

This fork should be based on the:

```text
original-gpu-unlock-only
```

branch of the upstream project.

Use your own GitHub fork for a reproducible installation.

Clone your fork's 8C58 branch:

```bash
git clone --branch omen-transcend-14-8c58 \
    https://github.com/lefarmer01600/5080-Unlock_Linux.git \
    5080-Unlock_Linux

cd ~/5080-Unlock_Linux
```

Replace:

```text
<YOUR_FORK_URL>
```

with the URL of your own GitHub fork.

For example, your repository should contain a dedicated branch similar to:

```text
omen-transcend-14-8c58
```

Do not use the upstream `main` branch for this hardware-specific setup.

---

# Verify the fork before installing

Check the thermal-profile configuration:

```bash
cat modprobe.d/omen_wmi_boost.conf
```

It should contain:

```text
# Installed by install.sh — max performance at load
options omen_wmi_boost persist=1 auto_boost=1 thermal_profile=0x31
```

The important value is:

```text
thermal_profile=0x31
```

not:

```text
thermal_profile=1
```

Check that the Smart Performance Gain modification exists in the module source:

```bash
grep -nE 'SET_POWER_LIMITS|concurrent_power_set|Smart Performance' \
    omen_wmi_boost/omen_wmi_boost.c
```

The source should contain the 8C58-specific power-budget implementation and the call that sets:

```text
Smart Performance Gain = 45
```

If those changes are not present, stop here and update your fork before installing.

---

# Build and install

First perform a non-destructive dry run:

```bash
./install.sh --dry-run
```

If the dry run looks correct:

```bash
sudo ./install.sh
```

The installer builds the module for the current kernel and installs the module, configuration, verification service, helper scripts, documentation, and kernel-install hook.

A reboot is recommended:

```bash
sudo reboot
```

---

# Verify after reboot

Check that the module is loaded:

```bash
lsmod | grep omen_wmi_boost
```

Check the firmware GPU state:

```bash
cat /sys/kernel/omen_wmi_boost/gpu_state
```

Expected:

```text
ctgp=1 ppab=1 dstate=1 ...
```

The exact `slowdown_temp` value may vary.

The important values are:

```text
ctgp=1
ppab=1
```

---

# Verify the 8C58 thermal profile

Run:

```bash
cat /sys/kernel/omen_wmi_boost/thermal_profile
```

Expected:

```text
0x31
```

This is important on the 8C58 platform.

If it says:

```text
0x01
```

the persistent configuration is wrong.

Check:

```bash
cat /etc/modprobe.d/omen_wmi_boost.conf
```

and make sure it contains:

```text
options omen_wmi_boost persist=1 auto_boost=1 thermal_profile=0x31
```

---

# Verify Smart Performance Gain

Run:

```bash
journalctl -k -b | grep -iE 'Smart Performance|omen_wmi_boost'
```

A successful boot should contain:

```text
Smart Performance Gain set to 45
```

This confirms that the additional HP power-budget command was accepted by the firmware.

---

# Verify NVIDIA power

Check:

```bash
nvidia-smi -q -d POWER
```

Or the important lines only:

```bash
nvidia-smi -q -d POWER | \
grep -E "Current Power Limit|Instantaneous Power Draw|Average Power Draw"
```

Do not use an idle reading as the final test.

The GPU may report approximately:

```text
Current Power Limit: 50 W
```

when idle or lightly loaded.

The correct test is under heavy GPU load.

---

# Test with Satisfactory

Start Satisfactory and verify that the game is using the RTX 4060:

```bash
nvidia-smi
```

The process list should contain something similar to:

```text
FactoryGameSteam-Win64-Shipping.exe
```

The RTX 4060 should show significant VRAM usage and GPU utilization.

For continuous monitoring:

```bash
watch -n 0.5 \
'nvidia-smi --query-gpu=power.draw,power.limit,clocks.gr,temperature.gpu,utilization.gpu,pstate --format=csv'
```

Or:

```bash
watch -n 0.5 \
'nvidia-smi -q -d POWER | grep -E "Current Power Limit|Instantaneous Power Draw|Average Power Draw"'
```

Under a sufficiently GPU-heavy workload, the expected result is:

```text
Current Power Limit : 65.00 W
```

Actual power draw will fluctuate.

For example:

```text
54 W
56 W
61 W
65 W
```

can all be normal depending on the workload and Dynamic Boost behavior.

---

# PRIME / NVIDIA GPU offloading

The laptop uses hybrid graphics.

The Intel Arc GPU normally drives the internal display, while individual applications can be offloaded to the NVIDIA GPU.

Check:

```bash
switcherooctl list
```

The NVIDIA device should be listed as the discrete GPU.

Test NVIDIA OpenGL offloading:

```bash
switcherooctl launch -g 1 glxinfo -B
```

The renderer should report the RTX 4060:

```text
OpenGL vendor string: NVIDIA Corporation
OpenGL renderer string: NVIDIA GeForce RTX 4060 Laptop GPU/PCIe/SSE2
```

For Steam/Proton games, use NVIDIA PRIME/Vulkan offloading when required.

---

# KDE Plasma / Wayland

This setup is intended to work with KDE Plasma Wayland.

You do not need to switch to X11 simply because the laptop uses NVIDIA Optimus.

Typical operation:

```text
Intel Arc
    ↓
KDE desktop / internal display

RTX 4060
    ↓
PRIME offload
    ↓
Games and GPU applications
```

---

# Battery behavior

Do not disable or blacklist the NVIDIA driver simply to save battery.

The RTX 4060 supports Runtime D3:

```bash
cat /proc/driver/nvidia/gpus/0000:01:00.0/power
```

Look for:

```text
Runtime D3 status: Enabled (fine-grained)
```

The PCI runtime power policy should remain:

```bash
cat /sys/bus/pci/devices/0000:01:00.0/power/control
```

Expected:

```text
auto
```

When the GPU has no clients and nothing requires the discrete GPU, it should eventually enter runtime suspend:

```bash
cat /sys/bus/pci/devices/0000:01:00.0/power/runtime_status
```

Expected when idle:

```text
suspended
```

When an application needs the NVIDIA GPU, it wakes automatically.

---

# External monitor behavior

On this laptop, an external monitor can keep the NVIDIA GPU active.

With no external monitor and no NVIDIA application running:

```text
RTX 4060 → can enter runtime suspend
```

With an external monitor attached:

```text
RTX 4060 → may remain active
```

This depends on how the particular display output is routed through the laptop.

On the tested system, the external monitor caused the NVIDIA GPU to remain active.

The external display also caused the GPU power limit to be exposed at:

```text
65 W
```

while the internal-display-only idle state could report:

```text
50 W
```

This does not mean that gaming on the internal display is limited to 50 W.

When a game was actually running on the internal display, the RTX 4060 successfully reached:

```text
65 W
```

through Dynamic Boost.

---

# KDE power profile

For gaming on AC:

```bash
powerprofilesctl set performance
```

Check:

```bash
powerprofilesctl get
```

Expected:

```text
performance
```

The KDE power profile does not permanently force the GPU to consume 65 W.

It establishes the laptop's high-performance platform policy. NVIDIA Dynamic Boost then dynamically allocates power between the CPU and GPU according to workload and firmware limits.

---

# After a Fedora kernel update

The module is out-of-tree, so every new kernel must have a module built for it.

First check the running kernel:

```bash
uname -r
```

Check the installed module:

```bash
modinfo omen_wmi_boost | grep filename
```

Check that it is loaded:

```bash
lsmod | grep omen_wmi_boost
```

Check:

```bash
cat /sys/kernel/omen_wmi_boost/gpu_state
```

Check:

```bash
cat /sys/kernel/omen_wmi_boost/thermal_profile
```

Expected:

```text
0x31
```

Then check the boot log:

```bash
journalctl -k -b | grep -iE 'omen_wmi_boost|Smart Performance'
```

You want:

```text
Smart Performance Gain set to 45
```

Finally test Satisfactory again:

```bash
nvidia-smi -q -d POWER | \
grep -E "Current Power Limit|Instantaneous Power Draw|Average Power Draw"
```

Under GPU load, the limit should be able to reach:

```text
65.00 W
```

The installer from the base project installs a kernel-install hook and rebuild helper intended to keep the module available across Fedora kernel updates.

---

# Troubleshooting

## Module is not loaded

Check:

```bash
lsmod | grep omen_wmi_boost
```

Check:

```bash
modinfo omen_wmi_boost
```

Check:

```bash
journalctl -k -b | grep -i omen_wmi_boost
```

If Secure Boot is enabled, verify that the module is trusted/signed.

---

## `gpu_state` does not show CTGP/PPAB

Run:

```bash
cat /sys/kernel/omen_wmi_boost/gpu_state
```

Then:

```bash
cat /sys/kernel/omen_wmi_boost/last_error
```

Check the kernel log:

```bash
journalctl -k -b | grep -iE 'omen_wmi_boost|CTGP|PPAB'
```

---

## Thermal profile is wrong

Run:

```bash
cat /sys/kernel/omen_wmi_boost/thermal_profile
```

It should be:

```text
0x31
```

Check:

```bash
cat /etc/modprobe.d/omen_wmi_boost.conf
```

It should contain:

```text
options omen_wmi_boost persist=1 auto_boost=1 thermal_profile=0x31
```

If the repository still contains:

```text
thermal_profile=1
```

the fork has not been updated correctly.

---

## Smart Performance Gain is missing

Check:

```bash
journalctl -k -b | grep -i "Smart Performance"
```

Expected:

```text
Smart Performance Gain set to 45
```

If the message is missing, verify that your fork's:

```text
omen_wmi_boost/omen_wmi_boost.c
```

contains the 8C58-specific `0x29` implementation.

---

## GPU is still around 50 W

First make sure the GPU is actually under load.

Check:

```bash
nvidia-smi
```

Then:

```bash
nvidia-smi -q -d POWER | \
grep -E "Current Power Limit|Instantaneous Power Draw|Average Power Draw"
```

Check:

```bash
systemctl status nvidia-powerd --no-pager
```

Check Dynamic Boost:

```bash
cat /proc/driver/nvidia/gpus/0000:01:00.0/power
```

Look for:

```text
Notebook Dynamic Boost: Supported
```

Then retry with a sufficiently GPU-heavy workload.

Do not assume that an idle 50 W limit means the unlock failed.

---

# Manual controls

Re-apply the complete performance sequence without rebooting:

```bash
echo 1 | sudo tee /sys/kernel/omen_wmi_boost/performance
```

Re-check:

```bash
cat /sys/kernel/omen_wmi_boost/gpu_state
```

Check the kernel log:

```bash
journalctl -k -b | tail -50
```

You should see the performance and Smart Performance operations being applied.

## Disable boost

While the module is loaded:

```bash
echo 0 | sudo tee /sys/kernel/omen_wmi_boost/boost
```

Use this only when you intentionally want to disable the GPU boost state.

## Do not use `modprobe -r` casually

The module depends on the shared Linux WMI subsystem.

On a running KDE system, unloading it can fail because other HP/WMI drivers are using the `wmi` module.

For example:

```text
modprobe: FATAL: Module wmi is in use.
```

A reboot is the safest way to change the module version.

---

# Uninstall

From the repository:

```bash
sudo ./install.sh --uninstall
```

The uninstaller removes the installed module and persistence configuration and disables the GPU boost state.

The repository checkout itself is not removed.

After uninstalling, reboot:

```bash
sudo reboot
```

---

# Files installed by the installer

The installer from the base project installs the following components:

```text
/lib/modules/<kernel>/updates/omen_wmi_boost.ko

/etc/modules-load.d/omen_wmi_boost.conf

/etc/modprobe.d/omen_wmi_boost.conf

/etc/systemd/system/omen-wmi-boost-verify.service

/usr/local/sbin/omen-wmi-boost-verify

/usr/local/sbin/omen-wmi-boost-rebuild

/etc/kernel/install.d/zz-omen-wmi-boost.install

/usr/share/doc/omen-wmi-boost/BOOT-SETUP.md
```

The source tree is also copied to:

```text
/usr/src/omen_wmi_boost
```

This allows the module to be rebuilt for a new kernel.

---

# Development

Build manually:

```bash
make -C omen_wmi_boost
```

Clean generated build files:

```bash
make -C omen_wmi_boost clean
```

Run the validation script:

```bash
sudo ./run-tests.sh
```

Inspect the source:

```text
omen_wmi_boost/omen_wmi_boost.c
```

Inspect the persistent module configuration:

```text
modprobe.d/omen_wmi_boost.conf
```

---

# 8C58-specific changes in this fork

This fork intentionally differs from the upstream `original-gpu-unlock-only` branch.

## Change 1 — OMEN v1 performance profile

The original configuration uses:

```text
thermal_profile=1
```

This fork uses:

```text
thermal_profile=0x31
```

because the target system board is:

```text
8C58
```

## Change 2 — Smart Performance Gain

The module has an additional HP WMI power-limit operation:

```text
Command: 0x29
```

with the 8C58 Smart Performance Gain value:

```text
45
```

The module logs:

```text
Smart Performance Gain set to 45
```

These two changes are required to reproduce the tested 8C58 configuration.

---

# Reinstalling Fedora

After a complete Fedora reinstall, the basic procedure is:

## 1. Install NVIDIA

Verify:

```bash
nvidia-smi
```

## 2. Install build dependencies

```bash
sudo dnf install gcc make kernel-devel-$(uname -r)
```

## 3. Enable NVIDIA Dynamic Boost support

```bash
sudo systemctl enable --now nvidia-powerd.service
```

## 4. Clone this fork

```bash
git clone --branch omen-transcend-14-8c58 <YOUR_FORK_URL> 5080-Unlock_Linux
cd ~/5080-Unlock_Linux
```

## 5. Verify the 8C58 configuration

```bash
cat modprobe.d/omen_wmi_boost.conf
```

It must contain:

```text
options omen_wmi_boost persist=1 auto_boost=1 thermal_profile=0x31
```

Verify the source contains the Smart Performance Gain modification:

```bash
grep -nE 'SET_POWER_LIMITS|concurrent_power_set|Smart Performance' \
    omen_wmi_boost/omen_wmi_boost.c
```

## 6. Build and install

```bash
./install.sh --dry-run
sudo ./install.sh
```

## 7. Reboot

```bash
sudo reboot
```

## 8. Verify

```bash
lsmod | grep omen_wmi_boost
cat /sys/kernel/omen_wmi_boost/gpu_state
cat /sys/kernel/omen_wmi_boost/thermal_profile
journalctl -k -b | grep -i "Smart Performance"
```

Expected:

```text
ctgp=1
ppab=1
thermal_profile=0x31
Smart Performance Gain set to 45
```

## 9. Test the GPU

Launch Satisfactory and run:

```bash
nvidia-smi -q -d POWER | \
grep -E "Current Power Limit|Instantaneous Power Draw|Average Power Draw"
```

Under a sufficiently heavy GPU workload:

```text
Current Power Limit: 65.00 W
```

---

# Known-good configuration

The tested configuration is:

```text
Laptop:
    HP OMEN Transcend Gaming Laptop 14-fb0xxx

Board:
    8C58

BIOS:
    F.10

GPU:
    NVIDIA GeForce RTX 4060 Laptop GPU

iGPU:
    Intel Meteor Lake-P Arc Graphics

Desktop:
    KDE Plasma / Wayland

Graphics:
    Hybrid / PRIME offload

NVIDIA:
    proprietary driver
    nvidia-powerd enabled

omen_wmi_boost:
    persist=1
    auto_boost=1
    thermal_profile=0x31

HP Smart Performance Gain:
    45

GPU power:
    50 W base
    up to 65 W under Dynamic Boost
```

---

# Safety

This project changes firmware-controlled power behavior.

Higher GPU power can result in:

* higher GPU and CPU temperatures
* higher fan speed
* more noise
* higher battery/AC power consumption
* additional thermal and electrical stress

Do not block the laptop's air intake or exhaust.

Monitor temperatures during long workloads.

The 65 W configuration is within the firmware's exposed power range on the tested hardware, but this project does not guarantee compatibility or safety on other models.

---

# Why this fork exists

The upstream project targets a different OMEN configuration.

This fork exists to preserve the configuration tested on:

```text
HP OMEN Transcend 14-fb0xxx
board 8C58
RTX 4060
```

The objective is not to provide a general OMEN tuning tool.

It is a reproducible Linux configuration for this specific platform.

---

# License

See `LICENSE`.

The kernel module follows the license specified in its source.

---

# Upstream project

This repository is derived from the `original-gpu-unlock-only` branch of the upstream `5080-Unlock_Linux` project.

The upstream project contains the original `omen_wmi_boost` implementation, installer, persistence support, verification service, and kernel-update rebuild support.

This fork adds the 8C58-specific changes documented above.
