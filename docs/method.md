# How the work runs

## Where Claude sits

Claude Code runs in the Claude desktop app on the Arch desktop, and its shell
is a real shell on that machine: same files, same logs, same hardware. It
reaches the server over SSH with a key. The server keeps no git checkout of its
own; config ships to it from the desktop's clone of the server repo, or arrives
through the server's own update bot (see [server.md](server.md)). For pages it
can't reach any other way, like the router's admin screen, it works in my
browser after I've logged in.

Each session starts with two files already loaded: the repo's `CLAUDE.md` and
the index of Claude's memory.

## One session, start to finish

On 2026-09-02 the desktop booted with three red `[FAILED]` lines about a
"Network Management" socket. I took a phone photo of the screen and asked what
they were. Claude read the photo, then checked the machine instead of the
internet:

```
systemctl --failed
journalctl -b -p err -g "Failed to listen"
systemctl is-enabled systemd-networkd.service
```

The journal had the full message that the boot screen cuts off:

```
systemd-networkd-varlink.socket: Socket service systemd-networkd.service not loaded, refusing.
```

File timestamps under `/etc/systemd/system` told the rest. On Aug 8, systemd's
presets had enabled every networkd socket. On Aug 11, networkd itself was
masked, because this machine runs NetworkManager instead. Only two units were
masked, so three newer sockets (added in systemd 258) kept trying to start a
service that was switched off. Impact: none, since `systemctl --failed` was
empty and the network was fine. The fix was one command for me to run:

```bash
sudo systemctl mask systemd-networkd-varlink.socket systemd-networkd-varlink-metrics.socket systemd-networkd-resolve-hook.socket
```

It also explained why `mask` and not `disable`: a later preset run can
re-enable a disabled unit, but not a masked one. About two minutes from photo
to fix. Then it wrote itself a memory note, so when a future systemd release
adds another unit like these, the answer is one `journalctl` search away
instead of a fresh investigation.

## Measure, then talk

A cause isn't stated until output shows it. When the desktop's VPN connection
failed with "Operation not supported," the first check was `uname -r` against
`ls /usr/lib/modules`. The nightly update had replaced the running kernel's
modules, so WireGuard couldn't load until a reboot. One comparison, and
nothing got reinstalled.

When something can't be measured (what I meant, which machine, whether a
device is still in use), the rule is to ask. That rule exists because guessing
went wrong; see [the mistakes](case-studies.md#when-claude-got-it-wrong).

## Prove it

Running isn't the same as working. The server's build day, 2026-08-20, ended
with each piece tested, not just started:

- **VPN.** Checked the tunnel's public IP, killed the tunnel, confirmed
  traffic stopped dead, confirmed it recovered.
- **DNS.** Confirmed ad blocking, then stopped AdGuard to prove the server
  itself could still resolve names without it.
- **Backups.** The next day, a restore drill pulled a service's config out of a
  snapshot (checksum-identical to the live file) and replayed the photo
  library's database dump into a scratch Postgres.

Alerts count as working once a bounce test reaches my phone: stop the service,
get the "down" push, start it, get the "up" push. That test exists because
build day missed something; see
[monitoring that paged nobody](case-studies.md#monitoring-that-paged-nobody).

The desktop works the same way. The GPU undervolt counts as a gain only
against same-day stock runs of the same benchmark: in Cyberpunk 2077 at 4K
with ray tracing, 66.56 → 68.37 fps average and 54.42 → 57.90 fps 1% lows.

## Git is the record

Each change is committed and pushed when it's made, then tagged and given a
GitHub Release with real notes. Commit messages carry the reasoning, so
`git log` explains the system as well as changing it.

Docs change in the same pass as whatever they describe. The server repo treats
its docs as a wiki that Claude writes and I direct, following
[Karpathy's LLM-wiki pattern](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f):
every fact has one home, pages describe the current state only, and history
lives in the changelog.

The server repo also has a deployment rule: work lands on a `develop` branch,
and merging a container change there *is* a deployment, because the server
pulls `develop` every morning and restarts whatever changed.

## Rules that stick

Two layers keep corrections from evaporating between sessions.

**`CLAUDE.md`** lives inside a repo and holds conventions: where a fact goes,
how docs are written, what a merge means. Claude Code reads it at the start of
every session. The server repo's version ends on the line that sums up the
whole approach: a correction that arrives twice is a rule that belongs in the
file. A starting point for a single machine can be this small:

```markdown
# CLAUDE.md

## This machine
- Arch Linux, KDE Plasma on Wayland, fish shell
- Timeshift snapshot before any system change

## Rules
- Show the output that proves a cause before proposing a fix.
- If something can't be checked, ask. Don't guess.
- Never print passwords, keys or tokens.
- Commit every config change with a message that says why.
- A correction that comes up twice becomes a rule in this file.
```

**Memory** lives outside any repo: a folder of short markdown notes Claude
writes for itself, plus an index with one line per note that loads into every
session. Notes come in four kinds: who I am, how I want things done
(feedback), what's in progress (project), and where things live (reference).
Feedback notes always have the same shape, the rule, then **Why**, then
**How to apply**, and the *why* is what lets Claude handle a case the rule
never anticipated. A note looks like this:

```markdown
---
name: which-machine
description: A request that fits both the desktop and the server gets one question first
metadata:
  type: feedback
---

When a request could belong to either machine (downloads, backups,
notifications), ask which one before building anything.

**Why:** a "download finished" alert was built end to end on the desktop when
the server was meant, and two tagged releases had to be undone.

**How to apply:** one question up front, never a plausible default.
```

Memory also keeps decisions made. An offsite backup, a boot splash, Usenet:
each was considered once, turned down, and written down as turned down, so
none of them gets pitched again a week later.

## The keys I keep

- **sudo.** Passwordless only for `pacman` and `timeshift`, which the 3 AM
  update needs. Everything else prompts for my password, so root commands come
  to me as a block to read and run myself.
- **The backup drive.** Anything that touches the server's backup disk is
  operator-only. Claude writes those steps down and never runs them.
- **Secrets.** `.env` files and keys never go into git. When a secret has to be
  checked, Claude compares lengths or hashes and never prints the value.
- **Logins.** I type the router and app passwords; Claude works in the session
  afterwards.
- **Anything public**, this repo included.

Claude Code's own permission system adds a layer: some commands, force-pushes
among them, are blocked outright. When a blocked step is genuinely needed,
Claude hands over one self-checking script instead. It refuses to run unless
the repo is in exactly the expected state, tags a rescue point before changing
anything, and verifies the result before it pushes. I read it and run it.

## Backups make it safe to try things

- **Desktop:** at 03:00, a Timeshift snapshot of the system, then a restic
  backup of `/home` to the server, then the update. If either backup fails,
  the update doesn't run. Manual updates go through the same gates.
- **Server:** restic at 04:00 every night to the 4 TB disk: photos, the file
  share, and every service's config. Movies and TV are single-copy on purpose,
  since they can be downloaded again.
