# PGST PG-108 2G — Great alarm panel, but a major firmware reliability flaw (reboots + corruption)

**Language:** **English (default)** | [Français](README.fr.md)

## Context
I genuinely like this alarm panel. In terms of compatibility it’s excellent (Tuya/Smart Life integration, sensors, etc.).  
However, I ran into a **major reliability issue**, to the point where I can’t trust it for security.

This repository contains my detailed report with evidence (photos + UART logs).

**Device:** PGST Alarm Host **PG-108 2G**  
**Alarm #1 (firmware):** `v1.25.03.n5` (UART)  
**Alarm #2 (replacement, firmware):** `v1.24.12.n5` (older, but same issue)

---

## What I observed (real usage)
### 1) Reboots / instability
I first noticed repeated restarts via Tuya debug logs, then dug deeper using UART.

**Security impact:**  
If a burglary happens during a reboot, the alarm may be ineffective.  
If the siren is active and the unit reboots, the siren may stop.

### 2) On-screen logs show clear signs of corruption
In the on-device logs (alarms / arming), I see impossible timestamps like:

- `65535-256-255 255:255:255`

And event labels appear in unrelated languages or corrupted characters, for example:
- `Péntek` (Hungarian)
- `Odłączenie zasilania` (Polish)
- `Telefonado … Online` (mixed / inconsistent)
- corrupted characters like `Шэ9?b?` (example in photos)

This looks like non-initialized / corrupted values in persistent storage.

### 3) Random voice prompts (alarm #1)
On the first unit, twice in ~4–5 months, the panel started speaking by itself in the middle of the night (no user interaction).

---

## The replacement unit also shows the issue
After my report, I received a replacement alarm panel.

- It runs an **older firmware** (`v1.24.12.n5`)
- After only 1–2 weeks, I can already see “corruption-like” entries in the on-screen logs
- Reboots still happen (and may even be more frequent)

So this does not look like an isolated faulty unit.

---

## UART evidence (1-minute capture)
The UART output includes:
- dev boot message with a typo: **`hello wrold.`**
- GSM modem polling loop: `AT+CSQ`, `AT+CGREG?` with `+CGREG: 0,0`
- repeated message that looks like a flash write routine being invoked:  
  `struParaRunning Flash Program End...`

A **redacted** excerpt is provided:
- `evidence/log_redacted_1min.txt`

Note: despite having access to UART, I could not communicate with the alarm (no echo / no interactive console), and I could not find any key combination to enter a “boot”/configuration mode.

---

## What I think is happening (hypothesis)
I can’t prove it 100% without dumping/sniffing flash activity, but the pattern strongly suggests:

- when GSM registration fails (`CGREG 0,0`), the firmware keeps polling in a loop;
- at the same time, it repeatedly triggers non-volatile writes (`Flash Program End` appears often).

Excessive flash writes could explain:
- corruption (impossible timestamps, mixed languages, weird characters);
- instability / reboots.

---

## Hardware info (PCB / components)
- PCB marking (silkscreen): `4G/2G_V1.0 — 2024-02-22`
- Main MCU: Nation **N32G455** (Cortex‑M)
- Wi‑Fi module: Tuya **CB3S** (BK7231N)
- External SPI flash: **BoyalMicro 25Q64ESSIG** (64 Mbits / 8 MB)

---

## Community help / feedback welcome
If you have:
- seen the same issue on **PGST PG-108 2G** (or a close variant),
- insight into what `struParaRunning Flash Program End...` means internally,
- successfully accessed **SWD** / extracted useful information from this board,
- or decoded the MCU↔Wi‑Fi (Tuya MCU) serial protocol,

please open an issue or share your findings.

**Legal note:** I’m only interested in analysis and reliability fixes on hardware I own. I do not intend to publish proprietary firmware binaries or help distribute copyrighted vendor code.
