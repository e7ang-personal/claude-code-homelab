# Case studies

Real problems from both machines. Each starts with the answer a quick search
tends to give, then what was actually going on. After those come the six bugs
that build day shook out, and Claude's own mistakes with the rule each one left
behind.

## Cyberpunk 2077 crashed in its own benchmark

**Symptom.** The in-game benchmark crashed partway through, on a setup that had
passed it the week before. The kernel log was clean.

**Quick answer.** Verify the game files, reinstall the GPU driver, try another
Proton build.

**What was going on.** All of those were tried and ruled out, along with game
settings, fresh Wine prefixes, Steam versus Lutris, sync modes, the filesystem
and the driver version. The break came from the boot menu: the same kernel
crashed when booted from GRUB and passed every time from a UKI entry. The only
meaningful difference was one kernel option. Arch turns on zswap (a compressed
cache in front of swap) by default, and this machine swaps to zram (compressed
swap in RAM), so swapped memory went through two layers of compression. The
crash dump pointed at a race condition in the game's own code, one it normally
wins. With both layers active on newer kernels, it started losing.

**Fix.** `zswap.enabled=0` on the kernel command line, the usual setting for a
zram system anyway. The benchmark passes.

## Downloads found no peers, but only some of them

**Symptom.** Torrents from public sources sat at zero connections while others
downloaded normally.

**Quick answer.** Forward a port, switch VPN servers, check the firewall.

**What was going on.** qBittorrent lives inside the VPN container's network,
behind a kill switch that drops anything not going through the tunnel. At
startup it had bound its sockets to the Docker network before the tunnel
interface existed. TCP traffic still found its way out through the tunnel, but
UDP trackers and DHT, which public torrents depend on, hit the kill switch and
died there. The kill switch was doing its job; the client was knocking on the
wrong door.

**Fix.** Bind qBittorrent to the tunnel interface, `tun0`. It rebinds by
itself whenever the tunnel comes back.

## The server dropped its drives mid-copy

**Symptom.** While 3 TB of media moved onto the new disk with a download
running, a drive went read-only. Later that evening it happened twice more, and
those times the whole box dropped off the network until it was power-cycled.

**Quick answer.** The drive is failing: run a disk check and replace it.

**What was going on.** The kernel log showed the USB Ethernet adapter resetting
in the same instant as the drives. Disk load can't reset a network adapter; a
sagging power supply can. Everything on the USB hub was browning out under
load.

**Fix.** The drives moved straight onto the laptop that night, then onto a dock
with its own 100 W supply. Nothing was lost across three brownouts, thanks to
`rsync --partial-dir` and ext4's journal.

## Monitoring that paged nobody

**Symptom.** None, which was the problem. Build day ended with every Uptime
Kuma monitor green.

**Quick answer.** Install Uptime Kuma and add your monitors, which is exactly
what had been done.

**What was going on.** The next day it turned out all 13 monitors had been
created without a notification channel. A dead service would have turned a
tile red on a dashboard nobody was watching, and nothing would have reached a
phone. Three days later the same gap turned up on five monitors added by hand.

**Fix.** The channel went onto every monitor, and a real bounce proved it: stop
a service, get the "down" push, start it, get the "up" push. New monitors get
the same test before they count. The script that seeds monitors into a fresh
install refuses to run unless the alert channel already exists.

## The backup drive that can't report its health

**Symptom.** The disk-health monitor read the 14 TB drive fine and got nothing
at all from the 4 TB backup drive.

**Quick answer.** Try `smartctl -d sat`, another cable, another USB port.

**What was going on.** Raw SCSI commands sent with `sg_raw` settled it. The
same ATA pass-through commands that worked on the 14 TB came back "Invalid
field in CDB" from the Seagate, in every variant. The USB bridge chip inside
the Seagate's own case refuses them, so no cable, port, dock or software flag
can fix it. Only taking the drive out of its enclosure would.

**Fix.** Stop chasing it, and watch its health another way: filesystem error
counts, kernel I/O errors, and restic's nightly check, which reads back a slice
of the backup data.

## A screen that never went to sleep

**Symptom.** With the Claude desktop app open, the OLED monitor never dimmed or
locked.

**Quick answer.** Check the power settings.

**What was going on.** The app was asking the desktop, over D-Bus, to stay
awake, and KWin honors that by freezing all of its idle timers. Turning off the
app's keep-awake setting fixed only part of it, because other parts of the app
send the same request. The first fix tried, a power-management setting, turned
out to be the wrong layer: it silenced one copy of the request while KWin kept
its own. It was reverted.

**Fix.** A small user service watches D-Bus for the app's inhibit requests and
releases each one immediately.

## Removing Flatpak would have removed the desktop

**Symptom.** None. This one was caught before it happened.

**Quick answer.** Remove Flatpak with pacman, then clear out orphans with
`pacman -Qtdq | sudo pacman -Rns -`.

**What was going on.** Checking the dependency chain first showed `flatpak` ←
`flatpak-kcm` ← `plasma-meta`. Taking out Flatpak meant taking out
`plasma-meta`, and that would have left its 55 Plasma packages (the window
manager, the desktop shell, System Settings and the rest) marked as orphans.
The next routine orphan cleanup would have deleted the desktop.

**Fix.** Mark those 55 packages as explicitly installed first
(`sudo pacman -D --asexplicit …`), then remove Flatpak. The orphan count
stayed at zero.

## Nightly backups that quietly stopped

**Symptom.** On some nights the desktop's 3 AM update-and-backup simply never
ran, with no error anywhere obvious.

**Quick answer.** Make sure the timer is enabled.

**What was going on.** It was enabled. The backups lived on an NTFS drive
shared with a Windows install, and Windows' Fast Startup leaves NTFS marked
dirty when it shuts down. Linux's `ntfs3` driver refuses to mount a dirty
volume, and the backup job requires that mount, so it never started and never
complained.

**Fix.** The backup repository moved to the server on 2026-09-01. Nothing on
the desktop's backup path depends on Windows now.

## Build day: six bugs, found by running the plan

The server was built from a written runbook and scripts that had been reviewed
beforehand. Running them for real still turned up six bugs, all fixed and
pushed:

- `sshd -T` reported a false failure under `set -o pipefail`.
- Ubuntu 26.04 ships the Rust rewrite of coreutils by default, and
  `install … /dev/stdin` fails under it.
- The VPN container's firewall had no exception for the home network, so
  qBittorrent's web page was unreachable from every browser.
- Jellyfin's discovery port (UDP 7359) was closed, so the Apple TV app couldn't
  find the server.
- The FlareSolverr image tag in the compose file didn't exist.
- Docker's bridge "hairpin": a container can't reach a port published on the
  host's own IP from the same bridge, which is why AdGuard needed a fixed
  container address.

## When Claude got it wrong

The method doesn't depend on Claude being right, because it isn't always. What
matters is that each mistake turns into a written rule.

| What happened | The rule now |
|---|---|
| Insisted on build day that its shell was a sandbox. It was my real desktop. | Treat every command as running on the real machine, because it is. |
| A sloppy shell pattern printed a VPN private key into the session log. The key was rotated. | Secrets get compared by length or hash; values are never printed. |
| Pushed a fix straight to `main` on the server repo, where `main` only takes releases. | Read the repo's branching rules before the first commit. |
| Built a "download finished" phone alert on the desktop when I meant the server. Two tagged releases had to be undone. | If a request could belong to either machine, ask which one first. |
| Told me three wrong things about the server repo from a local copy it hadn't refreshed. | `git fetch` before reasoning about branches. |
| Guessed the current version from the changelog and tagged a release lower than the real one. | Read the tags, not the prose. |
| Twice spun a story out of ambiguous log data instead of asking what I was seeing. | Ask, don't guess. |
| Ended answers with "want me to…?" instead of finishing obvious work. | Do the implied work, then report it. |
