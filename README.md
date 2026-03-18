# PGST PG-108 2G — Firmware design flaw: repeated reboots & likely excessive flash writes (UART evidence)

**Language:** **English (default)** | [Français](README.fr.md)

## Why this repo exists
I actually *love* this alarm panel: in terms of compatibility it’s excellent (Tuya/Smart Life integration, wide sensor support, etc.).  
Unfortunately, after a few months of real use I observed **critical stability issues** that make it unreliable for security use.

This repository documents:
- real-world symptoms (reboots, random voice prompts)
- UART logs as evidence
- technical analysis suggesting a **firmware design flaw**: tight modem polling combined with **frequent non-volatile writes** (flash/EEPROM), which can lead to wear/corruption and instability over time.

**Device:** PGST Alarm Host **PG-108 2G**  
**Firmware:** `FIRMWARE: v1.25.03.n5` (seen in UART log)

---

## Observed symptoms
### 1) Repeated reboots / instability
Sometimes the panel restarts repeatedly (in my case it can happen as often as ~hourly).

**Security impact:**  
During a reboot, the alarm may be temporarily ineffective. If a burglary happens at the same moment, this is a serious vulnerability.  
If the siren is active and the unit reboots, the siren may stop.

### 2) Random voice prompts at night
Twice in ~4–5 months, the panel started speaking by itself in the middle of the night with no user interaction.

---

## Evidence (UART log)
I first noticed frequent restarts while monitoring Tuya-related debug output. I then dug deeper using UART on the main MCU and captured logs.

The UART output shows:
- dev boot print with a typo: **`hello wrold.`**
- repeated GSM modem polling: `AT+CSQ`, `AT+CGREG?`
- `+CGREG: 0,0` indicating not registered (e.g., no SIM or no network)
- repeated lines strongly suggesting flash programming is being invoked:
  `struParaRunning Flash Program End, ...`

A 1-minute **redacted** log is included (recommended):
- `evidence/log_redacted_1min.txt`

Sensitive values should be removed:
- Wi‑Fi MAC
- IMEI (GSN)
- `magicCode`

---

## Technical analysis (hypothesis)
The log pattern suggests the firmware:
1. Polls the modem in a tight loop when registration fails (`CGREG 0,0`).
2. Repeatedly triggers flash programming / non-volatile commits (based on the recurring `Flash Program End` message).

Flash memory has limited write/erase endurance. Excessive write frequency can cause:
- configuration corruption (weird voice/time/locale)
- instability and reboots
- long-term device failure

Even if flash wear-out is not yet proven electrically, repeated “flash program end” messages during frequent polling is a strong reliability red flag.

---

## Suggested firmware fixes (for vendor)
- Rate-limit modem status checks (use backoff).
- Never write to flash on every poll/iteration.
- Commit only on change and at most every N minutes.
- Implement safer storage (journaling / wear leveling).
- Keep alarm core stable even when GSM is unavailable.

---

## Community help wanted / next steps
I first noticed frequent restarts while monitoring Tuya-related debug output. I then dug deeper using UART on the main MCU and captured the logs in this repo, which suggest a possible root cause (tight modem polling + frequent non-volatile commits).

**Next goal:** further confirm the hypothesis by correlating the UART messages with real flash write activity (SPI sniffing / flash dumps), and determine whether a firmware-side mitigation exists (rate limiting, storage strategy, etc.).

If anyone has:
- seen the same issue on **PGST PG-108 2G** (or close variants),
- successfully accessed **SWD** / dumped relevant flash contents,
- decoded the MCU↔Wi‑Fi (Tuya MCU) protocol on this model,
- or any insight into `struParaRunning Flash Program End...` and related storage routines,

please open an issue or share your findings.

**Note on legal/safety:** I’m only interested in analysis and reliability fixes on hardware I own. I do not intend to publish proprietary firmware binaries or help distribute copyrighted vendor code.
