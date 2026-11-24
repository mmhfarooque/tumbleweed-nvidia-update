# tumbleweed-nvidia-update

[![Release](https://img.shields.io/github/v/release/mmhfarooque/tumbleweed-nvidia-update)](https://github.com/mmhfarooque/tumbleweed-nvidia-update/releases/latest) [![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)

**One safe command to update an openSUSE Tumbleweed PC that has an NVIDIA GPU.**

On Tumbleweed a new kernel sometimes lands before the NVIDIA driver built for it.
Boot that kernel and you get a black screen, a fallback resolution, or no CUDA.
`update-everything` stops that from happening. It updates everything else as normal,
and it only moves the **kernel and the NVIDIA driver together**, once openSUSE has
published a driver built for that exact kernel.

📖 **Full explanation with diagrams:** [Tumbleweed + NVIDIA: Update the Kernel and Driver Together, Safely](https://mfaruk.com/blog/opensuse-tumbleweed-nvidia-safe-kernel-driver-update)

```
update-everything --check   # show what would change, change nothing
update-everything           # update everything safely
```

## What it does

| Step | What happens |
|---|---|
| 0. Pre-flight | Detects the NVIDIA driver branch (G06/G07) and the bootloader. Makes sure the kernel/NVIDIA zypper locks exist. Checks there is ≥ 5 GB free on `/` and ≥ 150 MB on the EFI partition. Refreshes the repos: if an official openSUSE repo fails it stops, if a third-party repo fails it warns and carries on. |
| 1. OS + apps | `zypper dup --no-allow-vendor-change` with a dry run first. The kernel and NVIDIA packages stay locked, and the script checks afterwards that none of them changed. Packman, Chrome and other third-party repos stay on their own vendor. |
| 2. Kernel + NVIDIA | `safe-kernel-update` installs the newest `kernel-default` only if the **official openSUSE OSS repo** has `nvidia-open-driver-<branch>-signed-kmp-default` built for that exact kernel (`…_k7.3.5_1-…`). The NVIDIA userspace packages (gl, compute, video…) move to the **same driver version**, or nothing happens. It takes a snapper pre/post snapshot and does a dry run first. It refuses if the update would remove packages. It checks that `nvidia.ko` exists and that its `modinfo` version matches, sets the default boot entry (systemd-boot) or checks the GRUB menu, and always puts the locks back. |
| 3. Flatpak | Updates system and user Flatpaks. |
| 4. Firmware | Runs `fwupdmgr` in **report-only** mode. BIOS/SSD firmware is never flashed automatically. |
| 5. Health | Checks that the newest kernel has an NVIDIA module, that the module version equals the userspace version, that the default boot entry is the newest kernel, that NVIDIA is loaded and that the locks are present. It also tells you whether a reboot is needed. |

Every step ends in a plain one-line status, then a final `SUCCESS` or `FAILED: <reason>`.
Logs are written to `/var/log/update-everything/`.

### Exit codes
`update-everything`: `0` success, `1` failed.
`safe-kernel-update`: `0` up to date, `10` waiting for the matching driver, `20` updated (reboot needed), `30` ready (`--check`), `1` failed.

## Requirements

- openSUSE **Tumbleweed** with snapper (the default btrfs install)
- NVIDIA driver from openSUSE: **`nvidia-open-driver-G06-signed-kmp-default`** or **`-G07-`**.
  This is the default for Turing/RTX 20-series and newer:
  `sudo zypper in nvidia-open-driver-G06-signed-kmp-meta` (or G07)
- Bootloader: systemd-boot or grub2-bls (via `sdbootutil`), or classic GRUB 2

### Which GPUs?

| GPU | Driver | Supported |
|---|---|---|
| RTX 20 / 30 / 40 / 50 series, GTX 16 series (Turing and newer) | openSUSE `nvidia-open-driver-G06/G07-signed-kmp` | ✅ Yes |
| GTX 10 series and older (Pascal, Maxwell, Kepler) | proprietary `nvidia-driver-G0x-kmp` / G05 / G04 from NVIDIA's repo | ❌ No: the script stops with a message and changes nothing |

Also not supported: the `kernel-longterm` flavour, and non-Tumbleweed releases (Leap, Slowroll).

## Install

```
git clone https://github.com/mmhfarooque/tumbleweed-nvidia-update.git
cd tumbleweed-nvidia-update
./update-everything --check
```

To install a fixed version, download it from [Releases](https://github.com/mmhfarooque/tumbleweed-nvidia-update/releases) or `git checkout v1.0.0`.

The first run asks for your sudo password and copies `update-everything` and `safe-kernel-update` to
`/usr/local/bin`. From then on just type `update-everything`. To pick up a new version, run `git pull`,
then `./update-everything --check` once.

On the first run it adds these zypper locks: `kernel-default*`, `nvidia-*`, `kernel-firmware-nvidia*`
(plus `kernel-devel`, `kernel-source`, `kernel-macros`, `kernel-syms` if you have them installed).
After that, a plain `zypper dup` or Discover can never move the kernel or the driver on its own.

## If something goes wrong

- The script tells you the snapper snapshot number and the rollback command: `sudo snapper rollback <N> && reboot`.
- Booted into a kernel with no NVIDIA driver? Pick the previous kernel in the boot menu, then run `update-everything --check`.
- `WAITING` is not an error. It means openSUSE hasn't published the matching driver yet, so your current kernel stays.

## Uninstall

```
sudo rm /usr/local/bin/update-everything /usr/local/bin/safe-kernel-update
sudo zypper rl 'kernel-default*' 'nvidia-*' 'kernel-firmware-nvidia*'
```

## Notes

- The NVIDIA KMP is not multiversion. After an update, **older kernels no longer have an NVIDIA module**.
  That's why the script sets the new kernel as the default boot entry.
- Override the detected driver branch with `DRV=G06 update-everything`.
- Plain bash plus the standard openSUSE tools (zypper, rpm, snapper, bootctl/sdbootutil or grub2, flatpak, fwupd). No extra dependencies.

## Disclaimer

Use at your own risk. `--check` never changes anything, so try that first.

| Path | Status |
|---|---|
| RTX 3060 · G07 620.70.06 · systemd-boot · `--check` and full run (everything up to date) | ✅ tested |
| Real kernel + driver update through v1.x of the script | ⏳ waiting for the next kernel release (the same logic was used by hand on 23 Nov 2026 for 7.3.5) |
| G06 driver | ⚠️ same logic, untested |
| Classic GRUB 2 | ⚠️ same logic, untested |

Tested a different setup? Please open an issue or PR with your result.

## Author

Created by **Mahmud Farooque** ([@mmhfarooque](https://github.com/mmhfarooque)).

License: MIT © 2026 Mahmud Farooque
