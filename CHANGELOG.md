# Changelog

## v1.0.0 — 2026-11-24
First release by Mahmud Farooque.
- `update-everything`: OS + apps (zypper dup), kernel + NVIDIA together, Flatpak, firmware report, health check; `--check` mode.
- `safe-kernel-update`: kernel + signed open NVIDIA KMP (G06/G07) only from official openSUSE repos, userspace pinned
  to the KMP driver version, snapper snapshots, dry run, module/boot-entry verification.
