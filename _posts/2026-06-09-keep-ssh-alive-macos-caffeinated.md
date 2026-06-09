---
layout: post
title: "Keeping SSH Alive on macOS — caffeinate, Keepalives, and What Actually Works"
date: 2026-06-09 21:05:20 +0700
categories: [DevOps, macOS]
tags: [ssh, macos, caffeinate, terminal, networking]
description: "How to stop SSH sessions from dropping on macOS using caffeinate and SSH keepalive settings. Covers why connections die, caffeinate deep-dive, ServerAliveInterval, and a one-liner that actually works."
---

You're tailing logs on a remote server. You go grab coffee. You come back. **Connection closed.** The terminal is frozen. Time to `ssh` again — lose your tmux session, your scrollback, your train of thought.

Two things kill SSH connections: **your Mac going to sleep**, and **idle TCP connections getting purged by firewalls and NAT gateways**. Here's how to fix both.

---

## Why SSH connections die

It's not one thing. It's a gauntlet:

1. **macOS sleep** — Close the lid or walk away, and the system naps. Sleeping Macs drop all TCP connections. When you wake them back up, the remote side has long since given up on the dead socket.

2. **NAT timeouts** — If you're behind a home router or coffee shop NAT, the gateway has a timer on idle mappings. 30 to 120 seconds of silence and your mapped port gets recycled. Next packet from the server? Nowhere to go. Dead.

3. **Stateful firewalls** — Corporate networks, cloud security groups, your ISP's CGNAT — all of them expire idle flows. Same result.

4. **TCP keepalives alone won't save you** — Default TCP keepalive kicks in after 2 hours on Linux. Your NAT gateway dropped you 1 hour 58 minutes ago.

---

## Part 1: Stop your Mac from napping

macOS ships with `caffeinate`, a built-in utility that creates *power assertions* to prevent sleep. No install needed. It's been there since Mavericks (10.9).

### Basic usage

```bash
# Keep system awake while a command runs
caffeinate ssh user@server

# Keep system awake for a duration
caffeinate -t 3600    # 1 hour

# Keep system awake indefinitely (Ctrl+C to stop)
caffeinate -i
```

### The assertions that matter

`caffeinate` can prevent specific types of sleep. The flags:

| Flag | What it prevents |
|------|------------------|
| `-i` | **Idle sleep** (system idle timer) |
| `-d` | **Display sleep** (screen off) |
| `-m` | **Disk sleep** |
| `-s` | **System sleep** (lid close, power button) |
| `-u` | **User activity assertion** (declares user is active) |

The two you care about for SSH:

```bash
# Idle sleep only (screen can dim, lid close still sleeps)
caffeinate -i ssh user@server

# Full prevention — screen stays on, lid close ignored (on AC power)
caffeinate -dis ssh user@server
```

### The reality check

`caffeinate -s` won't keep your Mac awake with the lid closed on **battery power**. Apple enforces lid-close sleep on battery as a hardware-level safety measure. If you need lid-closed SSH with no AC, you're fighting physics. Use `tmux`/`screen` on the remote side and accept that you'll reconnect.

For desktops and plugged-in laptops though, `caffeinate -i` is all you need.

---

## Part 2: Keep the TCP pipe alive

Even with your Mac wide awake, a silent SSH session is a ticking clock. Firewalls and NATs will kill the idle flow. The fix: make sure *something* moves through the pipe regularly.

### SSH-level keepalives

SSH has its own keepalive mechanism, independent of TCP:

```
# ~/.ssh/config (per host)
Host myserver
    HostName 192.168.1.100
    ServerAliveInterval 60
    ServerAliveCountMax 5
```

- **`ServerAliveInterval 60`** — Send a keepalive message every 60 seconds
- **`ServerAliveCountMax 5`** — Give up after 5 failed responses (5 minutes of silence)

This is an **encrypted SSH-level probe**, not a raw TCP packet. It actually traverses the encrypted channel, so it proves the remote sshd is still alive — not just that the kernel accepted a TCP ACK.

### Global defaults

Tired of configuring every host? Set it for all connections:

```
# ~/.ssh/config
Host *
    ServerAliveInterval 60
    ServerAliveCountMax 5
```

### The server side too

If you control the server, mirror it:

```
# /etc/ssh/sshd_config
ClientAliveInterval 60
ClientAliveCountMax 5
```

Then `sudo systemctl reload sshd`.

---

## Part 3: The one-liner

Putting it together:

```bash
caffeinate -i ssh -o ServerAliveInterval=60 -o ServerAliveCountMax=5 user@server
```

This:
- Prevents idle sleep (`caffeinate -i`)
- Sends SSH keepalives every 60s (`-o ServerAliveInterval=60`)
- Tolerates up to 5 minutes of silence before disconnecting

Bonus — wrap it in a shell alias:

```bash
# ~/.zshrc or ~/.bashrc
alias ssheep="caffeinate -i ssh -o ServerAliveInterval=60 -o ServerAliveCountMax=5"
```

Now `ssheep user@server` gives you a connection that actually survives your coffee break.

---

## Does `tmux` / `screen` help?

Yes, but for a different problem. `tmux` on the remote side means when your SSH *does* eventually die (and it will, eventually — networks are networks), you can reconnect and reattach without losing your session.

```bash
# On the server
tmux new -s work

# After reconnecting
tmux attach -t work
```

`tmux` is resilience. Keepalives are prevention. Use both.

---

## Quick decision tree

| Your scenario | What to do |
|---------------|------------|
| Laptop, on AC power, lid open | `caffeinate -i ssh ...` |
| Laptop, on AC power, lid closed | `caffeinate -s ssh ...` (and test it) |
| Laptop, on battery | `ssh -o ServerAliveInterval=60 ...` + `tmux` on remote |
| Desktop, always on | Just `ServerAliveInterval 60` in `~/.ssh/config` |
| Behind aggressive corporate firewall | `ServerAliveInterval 30` (30 seconds) |
| Everything | `tmux` on the remote side, always |

---

## References

- `man caffeinate` — macOS man page
- `man ssh_config` — `ServerAliveInterval` and `ServerAliveCountMax`
- [Apple's pmset documentation](https://developer.apple.com/library/archive/documentation/Darwin/Reference/ManPages/man1/pmset.1.html) — deeper power management
