# Fixing a Black Screen After Installing CachyOS on an MSI Stealth A16 AI+ (AMD + NVIDIA Hybrid Graphics)

> **Tested hardware:** MSI Stealth A16 AI+ with AMD Radeon 880M
> integrated graphics and NVIDIA GeForce RTX 4070 Laptop GPU\
> **Desktop tested:** XFCE / Xorg with LightDM\
> **Result:** AMD drives the internal display; NVIDIA remains available
> for PRIME render offload and runtime-suspends when idle.

## Quick Fix

If CachyOS installs successfully but boots to a black screen on an MSI
Stealth A16 AI+ with AMD/NVIDIA hybrid graphics, Xorg may be selecting
the NVIDIA GPU as the primary display device even though the laptop's
internal panel is connected to the AMD GPU.

Create:

``` text
/etc/X11/xorg.conf.d/20-amdgpu-primary.conf
```

with:

``` text
Section "OutputClass"
    Identifier "AMD Primary GPU"
    MatchDriver "amdgpu"
    Driver "amdgpu"
    Option "PrimaryGPU" "yes"
EndSection
```

Then reboot:

``` bash
sudo reboot
```

On the affected system, this eliminated the black screen while
preserving NVIDIA PRIME offloading.

**Do not blindly apply this fix to every black-screen problem.** Verify
that your laptop has a similar AMD/NVIDIA hybrid layout first using the
diagnostics below.

------------------------------------------------------------------------

## The Problem

CachyOS installed normally from the live ISO. The live environment
worked, the installation completed, and the bootloader worked. However,
after selecting the installed Linux system, the graphical environment
produced a black screen.

During troubleshooting, an early boot message appeared:

``` text
NVRM: No NVIDIA GPU found
```

That initially made the NVIDIA driver look like the likely culprit.

It wasn't.

The system contained:

-   AMD Radeon 880M integrated graphics
-   NVIDIA GeForce RTX 4070 Laptop GPU
-   NVIDIA open kernel modules
-   Xorg
-   LightDM

Both GPUs were ultimately found to be functional.

------------------------------------------------------------------------

## Step 1 --- Get a Text Console

At the CachyOS boot menu, highlight the normal Linux entry and press
`e`.

Append this to the kernel command line:

``` text
systemd.unit=multi-user.target
```

Boot the edited entry with **Ctrl+X** or **F10**.

This prevents the display manager from starting and gives you a text
login where the graphics stack can be inspected safely.

### Optional AMD-only diagnostic boot

If needed, temporarily blacklist the NVIDIA modules:

``` text
module_blacklist=nvidia,nvidia_drm,nvidia_modeset,nvidia_uvm
```

On this system, blacklisting NVIDIA allowed the graphical desktop to
work. That was an important clue, but it was **not** used as the
permanent solution.

------------------------------------------------------------------------

## Step 2 --- Verify the GPUs

Run:

``` bash
lspci -nnk | grep -EA3 'VGA|3D|Display'
```

The affected machine showed both GPUs.

The AMD GPU used:

``` text
Kernel driver in use: amdgpu
```

The RTX 4070 used:

``` text
Kernel driver in use: nvidia
```

With NVIDIA enabled, also check:

``` bash
nvidia-smi
```

Despite the earlier `NVRM: No NVIDIA GPU found` messages, `nvidia-smi`
successfully detected the RTX 4070.

This was an important lesson: **an early NVIDIA error did not mean the
GPU was actually unavailable once the system finished initializing.**

------------------------------------------------------------------------

## Step 3 --- Separate the NVIDIA Driver From the Graphical Login

Boot with:

``` text
systemd.unit=multi-user.target
```

but **without** the NVIDIA blacklist.

Then run:

``` bash
nvidia-smi
```

On this machine, the RTX 4070 worked correctly from the text console.

That proved:

-   The GPU existed.
-   The NVIDIA kernel driver loaded.
-   The driver could communicate with the GPU.
-   The failure occurred when the graphical stack started.

This prevented unnecessary driver reinstalls and kernel swapping.

------------------------------------------------------------------------

## Step 4 --- Start LightDM Manually

From the text console:

``` bash
sudo systemctl start lightdm
```

The screen went black, but LightDM itself remained healthy.

From another TTY (for example **Ctrl+Alt+F3**):

``` bash
systemctl status lightdm --no-pager -l
```

LightDM was active and Xorg was running.

Xorg also did not report a conventional fatal `(EE)` error.

That pointed toward a display-routing/configuration issue rather than
Xorg simply crashing.

------------------------------------------------------------------------

## Step 5 --- Inspect Xorg

Useful commands include:

``` bash
grep -Ei 'amdgpu|nvidia|primary|modeset|output|connected|screen' \
    /var/log/Xorg.0.log | tail -100
```

and:

``` bash
grep -Ei 'OutputClass|Primary GPU|PrimaryGPU|Screen 0|NVIDIA\(0\)|AMDGPU\(G0\)' \
    /var/log/Xorg.0.log | head -100
```

The critical clues were similar to:

``` text
NVIDIA(0): No enabled display devices found
NVIDIA(0): AllowEmptyInitialConfiguration is enabled
NVIDIA(0): Setting mode "NULL"
NVIDIA(0): Virtual screen size determined to be 640 x 480
```

At the same time, the AMD GPU was detecting the laptop's internal eDP
panel and its EDID/modes.

In other words:

``` text
NVIDIA(0)  -> being treated as the primary Xorg GPU
AMDGPU(G0) -> actually connected to the internal laptop panel
```

The NVIDIA GPU was being made Screen 0 even though it had no active
display attached to it.

That explained why Xorg and LightDM could technically be running while
the laptop screen remained black.

------------------------------------------------------------------------

## Step 6 --- Check the Xorg OutputClass Configuration

Search both the administrator configuration and package-provided
configuration:

``` bash
grep -RniE 'nvidia|amdgpu|BusID|PrimaryGPU|AllowEmptyInitialConfiguration' \
    /etc/X11 /usr/share/X11/xorg.conf.d 2>/dev/null
```

The NVIDIA package provided an OutputClass configuration under:

``` text
/usr/share/X11/xorg.conf.d/
```

which included NVIDIA configuration and
`AllowEmptyInitialConfiguration`.

Do **not** edit the package-owned file directly. A future package update
can replace it.

Instead, create a local override.

------------------------------------------------------------------------

## Step 7 --- Make AMD the Primary Xorg GPU

Create:

``` bash
sudo mkdir -p /etc/X11/xorg.conf.d
sudo nano /etc/X11/xorg.conf.d/20-amdgpu-primary.conf
```

Add:

``` text
Section "OutputClass"
    Identifier "AMD Primary GPU"
    MatchDriver "amdgpu"
    Driver "amdgpu"
    Option "PrimaryGPU" "yes"
EndSection
```

Save the file and restart LightDM:

``` bash
sudo systemctl restart lightdm
```

or reboot:

``` bash
sudo reboot
```

On this MSI Stealth A16 AI+, this was the actual fix.

The desired architecture became:

``` text
Internal eDP display
        |
        v
AMD Radeon 880M
        |
        v
    Xorg / Desktop
        |
        +---- PRIME render offload ----> NVIDIA RTX 4070
```

------------------------------------------------------------------------

## Step 8 --- Verify Normal Rendering Uses AMD

From a terminal **inside your graphical desktop as your normal user**,
run:

``` bash
glxinfo | grep "OpenGL renderer"
```

On the working system:

``` text
OpenGL renderer string: AMD Radeon 880M Graphics ...
```

If `glxinfo` is missing, install the package providing the Mesa
utilities for your system.

------------------------------------------------------------------------

## Step 9 --- Verify NVIDIA PRIME Offload

Run:

``` bash
prime-run glxinfo | grep "OpenGL renderer"
```

The renderer should change to the NVIDIA GeForce RTX 4070 Laptop GPU.

This confirms that making AMD primary does **not** disable the NVIDIA
GPU. The RTX can still be used for applications that need it.

For example:

``` bash
prime-run <application>
```

can be used to launch an application on NVIDIA.

------------------------------------------------------------------------

## Step 10 --- Verify NVIDIA Runtime Power Management

The RTX 4070 should ideally suspend when unused.

For this machine the NVIDIA PCI address was:

``` text
0000:c4:00.0
```

Your address may be different. Determine it with `lspci` rather than
copying this value blindly.

Check:

``` bash
cat /sys/bus/pci/devices/0000:c4:00.0/power/control
cat /sys/bus/pci/devices/0000:c4:00.0/power/runtime_status
```

The working system reported:

``` text
auto
suspended
```

NVIDIA's parameters included:

``` text
DynamicPowerManagement: 3
```

and `nvidia-persistenced` was disabled/inactive.

### Why `nvidia-smi` can be misleading for idle power

`nvidia-smi` may show the GPU awake at roughly 11 W even when runtime
power management is working.

Running `nvidia-smi` itself wakes the GPU so that it can be queried.

The kernel's:

``` text
power/runtime_status
```

is therefore useful when determining whether the GPU actually returns to
runtime suspend.

------------------------------------------------------------------------

## Optional --- XFCE + LightDM

The final system was switched to XFCE.

Install XFCE:

``` bash
sudo pacman -S xfce4 xfce4-goodies
```

If LightDM does not show a session selector, edit:

``` text
/etc/lightdm/lightdm.conf
```

Under:

``` text
[Seat:*]
```

set:

``` text
user-session=xfce
```

The graphics fix itself is independent of XFCE, but XFCE/Xorg was the
final tested configuration.

------------------------------------------------------------------------

## Things That Did NOT Fix the Root Cause

Several paths were investigated before identifying the Xorg routing
issue:

-   Reinstalling was unnecessary.
-   The SSD was not the problem.
-   `nomodeset` was useful only as a diagnostic idea.
-   Switching to the LTS kernel did not solve it.
-   Permanently blacklisting NVIDIA would have hidden the problem but
    sacrificed the RTX 4070.
-   The NVIDIA kernel packages were already installed and matched the
    CachyOS kernels.
-   The early `NVRM: No NVIDIA GPU found` message was not the final
    state of the GPU.

The decisive evidence was:

1.  AMD-only graphics worked.
2.  NVIDIA worked from a text console.
3.  `nvidia-smi` detected the RTX 4070.
4.  LightDM and Xorg remained running during the black screen.
5.  Xorg configured `NVIDIA(0)` with no display while AMD detected the
    internal eDP panel.
6.  Explicitly setting AMD `PrimaryGPU` fixed the display while
    retaining PRIME offload.

------------------------------------------------------------------------

## If You Are Troubleshooting Similar Hardware

Don't assume this exact override is appropriate merely because you have
an MSI laptop or an NVIDIA GPU.

The important question is:

**Which GPU is physically connected to the internal display, and which
GPU is Xorg treating as primary?**

If your logs show the same pattern---NVIDIA as Screen 0 with no enabled
display devices while AMD detects the internal panel---this workaround
may be relevant.

Useful information to include when asking for help:

``` bash
uname -r
lspci -nnk | grep -EA3 'VGA|3D|Display'
nvidia-smi
systemctl status display-manager --no-pager
grep -Ei 'NVIDIA\(0\)|NVIDIA\(G0\)|AMDGPU\(0\)|AMDGPU\(G0\)|PrimaryGPU|Screen 0' \
    /var/log/Xorg.0.log
```

------------------------------------------------------------------------

## Short Version

For this specific MSI Stealth A16 AI+:

**Problem**

``` text
NVIDIA RTX 4070 -> Xorg Screen 0 -> no display attached -> black screen
AMD Radeon 880M -> internal eDP panel -> not primary
```

**Fix**

``` text
/etc/X11/xorg.conf.d/20-amdgpu-primary.conf
```

``` text
Section "OutputClass"
    Identifier "AMD Primary GPU"
    MatchDriver "amdgpu"
    Driver "amdgpu"
    Option "PrimaryGPU" "yes"
EndSection
```

**Result**

``` text
AMD Radeon 880M -> internal display + normal desktop
NVIDIA RTX 4070 -> PRIME offload + runtime suspend when idle
```

------------------------------------------------------------------------

*This guide documents one real troubleshooting case and its working
configuration. Firmware revisions, GPU variants, display routing,
desktop environments, and future driver/Xorg changes can alter the
behavior, so verify your own hardware and logs before applying the fix.*
