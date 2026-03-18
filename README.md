# PGST PG-108 2G — Great device, but a serious firmware reliability flaw (reboots + data corruption)

**Language:** **English (default)** | [Français](README.fr.md)

## Context
I genuinely like this alarm panel. In terms of compatibility it’s excellent (Tuya/Smart Life integration, wide sensor support, etc.).  
But I ran into a **major reliability issue** that makes it unsafe to trust as a security system.

This repo is my detailed report with evidence (photos + UART logs).

**Device:** PGST Alarm Host **PG-108 2G**  
**Unit #1 firmware:** `v1.25.03.n5` (UART)  
**Replacement unit firmware:** `v1.24.12.n5` (older, still affected)

---

## What I observed (real usage)
### 1) Reboots / restarts happen repeatedly
I started noticing repeated restarts in Tuya debug logs, then confirmed it by investigating deeper (UART).

**Security impact:**  
If a burglary happens during a reboot window, the system may be ineffective.  
Also, if the siren is sounding and the unit reboots, the siren can stop.

### 2) UI / logs show strong signs of corruption
On the device screen (alarm/arming history), I see impossible timestamps like:

- `65535-256-255 255:255:255`

And event names randomly appear in unrelated languages or corrupted characters, e.g.:
- `Péntek` (Hungarian)
- `Odłączenie zasilania` (Polish)
- `Telefonado … Online` (mixed / inconsistent)
- corrupted characters like `Шэ9?b?` (example from photo)

This looks like corrupted/non-initialized values in persistent storage.

### 3) Random voice prompts (Unit #1)
On the first unit, twice in ~4–5 months, the panel spoke by itself in the middle of the night (no user interaction).

---

## Replacement unit is also affected
After reporting the problem, I received a replacement device.

- The replacement runs an **older firmware** (`v1.24.12.n5`)
- After only 1–2 weeks, I already see similar “corruption-like” entries in the on-device logs
- Restarts still happen (and may be even more frequent)

So this does not look like a single faulty unit.

---

## UART evidence (1-minute capture)
UART output includes:
- dev boot message with typo: **`hello wrold.`**
- GSM modem polling loop: `AT+CSQ`, `AT+CGREG?` with `+CGREG: 0,0`
- repeated message suggesting flash programming is invoked:
  `struParaRunning Flash Program End...`

A recommended, **redacted** excerpt is included:
- `evidence/log_redacted_1min.txt`

(Please redact MAC / IMEI / magicCode before publishing.)

---

## What I think is happening (hypothesis)
I can’t prove it 100% without dumping/sniffing flash activity, but the pattern strongly suggests:

- when GSM registration fails (`CGREG 0,0`), firmware keeps polling in a loop  
- at the same time, it repeatedly triggers non-volatile writes (“Flash Program End” appears many times)

Excessive flash writes can cause wear/corruption over time and explain:
- corrupted timestamps / mixed languages / random characters
- instability and reboots

---

## Community help / if you have experience with this model
If you have:
- the same issue on **PGST PG-108 2G** (or a close variant),
- info about what `struParaRunning Flash Program End...` really means internally,
- experience accessing **SWD** / dumping flash contents on this board,
- or you already reversed the MCU↔Wi‑Fi (Tuya MCU) serial protocol,

please open an issue or share details.

**Important note:** I’m only interested in analysis and reliability fixes on hardware I own. I do not intend to publish proprietary firmware binaries or help distribute copyrighted vendor code.
