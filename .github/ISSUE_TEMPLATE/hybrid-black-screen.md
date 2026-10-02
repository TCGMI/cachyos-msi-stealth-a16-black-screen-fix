---
about: Report similar AMD/NVIDIA hybrid-graphics behavior
labels: troubleshooting
name: Similar black-screen issue
title: "[Hardware](#hardware)"
---

## Hardware

**Laptop/model:**\
**CPU:**\
**Integrated GPU:**\
**Discrete GPU:**\
**MUX / graphics mode, if known:**

## Software

**CachyOS/kernel version:**\
**Desktop environment:**\
**Display manager:**\
**Xorg or Wayland:**\
**NVIDIA driver version:**

## Symptoms

Describe what happens after boot and when the graphical environment
starts.

## Diagnostic output

Please remove unrelated identifying information before posting logs.

``` text
uname -r
```

``` text
lspci -nnk | grep -EA3 'VGA|3D|Display'
```

``` text
nvidia-smi
```

``` text
systemctl status display-manager --no-pager
```

For Xorg:

``` text
grep -Ei 'NVIDIA\(0\)|NVIDIA\(G0\)|AMDGPU\(0\)|AMDGPU\(G0\)|PrimaryGPU|Screen 0' /var/log/Xorg.0.log
```

## Did the AMD PrimaryGPU override help?

-   [ ] Yes
-   [ ] No
-   [ ] Not tested
-   [ ] Not applicable

## Additional notes

Include relevant BIOS/firmware changes, kernel parameters, or
troubleshooting already attempted.
