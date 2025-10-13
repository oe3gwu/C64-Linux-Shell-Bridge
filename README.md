# C64 ↔ Linux Shell Bridge (README)

Short, focused documentation for a retro-computing bridge: a Commodore-64 using a Wi-Fi modem and **StrikeTerm** connects (raw TCP) to a Linux host that provides a VT100-style interactive shell session.  

**Important security note:** the original, unauthenticated pattern (listening on a network port and spawning a shell) is **inherently dangerous**. This README documents behavior and safe alternatives; it does **not** instruct how to expose an unauthenticated shell to untrusted networks. Use the hardened example and the mitigations below — and always keep vintage systems on isolated test networks.

---

## 1 — What this documents

This file documents three things at a high level:

- What the Linux-side listener (systemd + socat) *does conceptually*.
- How the C64 (StrikeTerm + Wi-Fi modem) should connect (client-side configuration).
- Security considerations, mitigations, and troubleshooting for a safe retro setup.

This README purposely avoids step-by-step instructions that would expose unauthenticated shells on public networks. If you are the system owner and require a runnable unit for a secured, local-only setup, see the **Hardened Example Service** section below.

---

## 2 — Conceptual overview (what happens)

1. The C64 runs StrikeTerm and sends/receives bytes to/from a serial-attached Wi-Fi modem.
2. The modem joins a Wi-Fi LAN and forwards serial bytes to/from a host:port via **raw TCP** (no telnet negotiation).
3. On the Linux host, a listener accepts the TCP connection, allocates a pseudo-terminal (pty), and attaches a shell process to that pty, presenting a simple VT100/VT102 compatible session to the remote client.
4. The user on the C64 sees a shell prompt and can interact with the system as if at a terminal.

> Note: the listener/spawn-shell pattern grants shell access to any party who can connect to that port. That is why network exposure must be strictly controlled.

---

## 3 — StrikeTerm / C64 (client) — minimal required settings

These are client-side settings you will configure in StrikeTerm (or equivalent) when using a Wi-Fi modem bridging serial → TCP:

- **Connection type:** `Raw TCP` (avoid telnet mode)  
- **Host / Port:** the IP and port of the Linux host (match the listener)  
- **Terminal type / TERM:** `vt100` (or `vt102`)  
- **Local echo:** **Off** (the remote shell will echo)  
- **Line ending / Newline:** `CR` for outgoing newlines is commonly expected by many C64 programs — adjust if needed.  
- **Serial params (C64 ↔ modem):** set to match your modem (typical: `2400`/`4800`/`9600` baud, `8N1`, no flow control).

**Why raw TCP?** Vintage C64 terminal programs frequently cannot interpret Telnet option negotiation bytes (IAC 0xFF sequences). A raw TCP bridge forwards only the user data bytes and avoids inserting telnet negotiation sequences that would appear as garbage on the C64.

---

## 4 — Hardened example service (safe pattern)

Below is a **safer** example of a systemd service unit that runs a socat-based relay but **binds to loopback only** and includes some systemd hardening directives. This example is intended as a local-only, lab-safe pattern — use it only on trusted hosts and networks or with additional wrapping (SSH tunnels, TLS) for remote access.

> This is an example for *local testing* and audit purposes. Do **not** copy a service that listens on `0.0.0.0` on an Internet-facing host.

```ini
# /etc/systemd/system/c64-socat-listener-local.service
[Unit]
Description=C64 socat shell listener (local-only, lab/demo)
After=network.target

[Service]
# Run under a dedicated unprivileged user (create user 'c64shell' separately)
User=c64shell
Group=c64shell

# Bind to loopback only to require an SSH tunnel or local access
ExecStart=/usr/bin/socat \
  TCP-LISTEN:2323,bind=127.0.0.1,reuseaddr,fork \
  EXEC:'/usr/bin/env -i TERM=vt100 PS1="$ " /bin/sh -i',pty,setsid,ctty,icanon=1,echo=1,echoctl=0,isig=1,iexten=1,opost=1,onlcr=1,icrnl=1

Restart=on-failure
RestartSec=5

# Hardening recommendations
ProtectSystem=full
ProtectHome=yes
NoNewPrivileges=yes
PrivateTmp=yes
ProtectKernelTunables=yes
ProtectKernelModules=yes
ProtectControlGroups=yes
RestrictAddressFamilies=AF_UNIX AF_INET AF_INET6
CapabilityBoundingSet=CAP_CHOWN CAP_SETGID CAP_SETUID
StateDirectory=c64-socat-listener

[Install]
WantedBy=multi-user.target
