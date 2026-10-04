# The desktop

Ryzen 5 5600X, RTX 5070 Ti, a 4K 120 Hz OLED. Arch Linux running CachyOS's
optimized repos and kernel, KDE Plasma on Wayland, NVIDIA's open kernel
modules, and fish as the shell. Installed 2026-08-07.

## One script rebuilds it

`bootstrap.sh`, 1,322 lines, runs from the official Arch live USB and
reproduces this machine: partitions, CachyOS repos, the CachyOS kernel with a
stock-kernel fallback, GRUB, NVIDIA drivers when it finds an NVIDIA card,
Plasma, the gaming stack, the backup automation and the dotfiles. It asks only
what it can't safely work out for itself (the target disk with a typed
confirmation, passwords, and optional Wi-Fi and backup credentials) and detects
the rest, such as CPU vendor, GPU and x86-64-v3 support. If a run gets
interrupted, it recognizes its own partitions and offers to resume. It was
proven in a throwaway virtual machine on 2026-08-16 before it was trusted, and
no secrets live in its repo.

## Updates that can't strand me

`daily-pac` runs at 03:00, or at the next boot if the PC was off:

1. Timeshift snapshots the system.
2. Restic backs up `/home` to the server.
3. Only then, the update.

If step 1 or 2 fails, step 3 doesn't happen. Typing `pac -Syu` runs the same
three gates by hand. Restores land in `~/Restore` and never overwrite live
files.

The config behind it lives in a repo as real files. Shell functions are
symlinked into place, so an edit is live in the next shell. Root-owned files
like the sudoers rule and the systemd units are copied, because sudo won't
trust a file it doesn't own. A lint script (`fish -n`, `visudo -c`,
`systemd-analyze verify`) runs before every commit.

## Gaming

- Lutris with umu-launcher and GE-Proton runs 12 Windows games. All game data
  sits on its own 2 TB drive, and `$HOME` holds only apps.
- Every game's config pins an exact Proton version. The "latest" alias changes
  underneath you, and in Lutris it was silently falling through to Proton
  Experimental for every game that used it.
- The GPU is tuned with LACT, because NVIDIA's own overclocking settings need
  X11. An MSI Afterburner profile that ran on Windows for a year was translated
  one curve point at a time; against same-day stock runs it's worth +2.7%
  average and +6.4% 1% lows in Cyberpunk 2077 at 4K with ray tracing.
- DLSS model overrides are set per game through DXVK-NVAPI environment
  variables, and confirmed in NVIDIA's own logs rather than assumed from a
  settings screen.
- HDR toggles on a hotkey, the PS5 controller powers itself off after 30 idle
  minutes, and the tiling window manager leaves games alone.

## Smaller things

- **wincalc:** a Windows 11–style calculator written from scratch in PyQt6,
  with tests.
- **oled-guard:** stops the Claude app from keeping the OLED awake
  ([case study](case-studies.md#a-screen-that-never-went-to-sleep)).
- **Flatpak, removed** without taking Plasma with it
  ([case study](case-studies.md#removing-flatpak-would-have-removed-the-desktop)).
