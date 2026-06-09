# VMTA Gateway — Modbus TCP Diagnostic Register Map

The gateway exposes a Modbus TCP server for real-time health monitoring from SCADA systems (Citect, Elipse, Ignition, etc.).

**Default:** host `0.0.0.0`, port `502`, unit ID `1`  
Configure in `config.yaml` under `diagnostics.modbus`.

---

## Discrete Inputs — Function Code 02 (Read)

Health bits. `1` = OK / healthy, `0` = fault or disabled.

| Address | Name | Description |
|---|---|---|
| 0 | System | Always `1` while the gateway process is running |
| 1 | License | `1` = valid license key, `0` = demo mode active |
| 2 | Modem | `1` = COM port open and AT emulator responding |
| 3 | SMTP Gateway | `1` = local SMTP server (port 2525) accepting connections |
| 4 | SCADA Engine | `1` = alarm filter engine running |
| 5 | SCADA Source | `1` = alarm file or serial port being actively read |
| 6 | Notification OK | `1` = no errors, `0` = last notification delivery failed (latched) |
| 7 | Twilio SMS | `1` = channel enabled and healthy |
| 8 | Telegram | `1` = channel enabled and healthy |
| 9 | SMTP Outbound | `1` = outbound email channel healthy |
| 10 | Twilio WhatsApp | `1` = channel enabled and healthy |

> Disabled components always report `1` (not a fault).  
> Address 6 is latched — it stays `0` until reset via Coil 100.

---

## Coils — Function Code 01 (Read) / 05 (Write Single)

Write `1` to trigger an action. The gateway clears the coil automatically after processing.

| Address | Name | Description |
|---|---|---|
| 100 | Reset Faults | Write `1` to clear the latched notification error (DI address 6) |
| 101 | Send On-Request Message | Write `1` to immediately dispatch the configured on-request message to all destinations |

---

## Holding Registers — Function Code 03 (Read)

| Address | Name | Type | Range | Description |
|---|---|---|---|---|
| 0 | Status Word | UINT16 | 0–2047 | Bitmask of all 11 discrete inputs packed into bits 0–10 (see below) |
| 1 | Total Alarms | UINT16 | 0–65535 | Total alarms processed since startup — wraps at 65535 |
| 2 | Last Alarm Timestamp High | UINT16 | — | High 16 bits of Unix timestamp of last alarm |
| 3 | Last Alarm Timestamp Low | UINT16 | — | Low 16 bits of Unix timestamp of last alarm |
| 4 | Status Code | UINT16 | 1–3 | `1` = OK, `2` = Warning (notification error), `3` = Fault |
| 5–9 | *(reserved)* | — | — | — |
| 10–19 | Status Message | 10 × UINT16 | — | ASCII string, 2 chars per register, space-padded (20 chars total) |
| 20 | Last Alarm Year | UINT16 | e.g. 2026 | Year component of last alarm time |
| 21 | Last Alarm Month | UINT16 | 1–12 | Month component |
| 22 | Last Alarm Day | UINT16 | 1–31 | Day component |
| 23 | Last Alarm Hour | UINT16 | 0–23 | Hour component |
| 24 | Last Alarm Minute | UINT16 | 0–59 | Minute component |
| 25 | Last Alarm Second | UINT16 | 0–59 | Second component |

### Status Word bitmask (HR 0)

| Bit | Component |
|---|---|
| 0 | System |
| 1 | License |
| 2 | Modem |
| 3 | SMTP Gateway |
| 4 | SCADA Engine |
| 5 | SCADA Source |
| 6 | Notification OK |
| 7 | Twilio SMS |
| 8 | Telegram |
| 9 | SMTP Outbound |
| 10 | Twilio WhatsApp |

Example: `0x07FF` (2047) = all 11 bits set = all components healthy.

### Reading the 32-bit timestamp (HR 2–3)

```python
timestamp = (hr[2] << 16) | hr[3]
from datetime import datetime
dt = datetime.fromtimestamp(timestamp)
```

### Reading the Status Message (HR 10–19)

Each register holds 2 ASCII characters (high byte = first char, low byte = second char):

```python
message = ""
for reg in hr[10:20]:
    message += chr((reg >> 8) & 0xFF) + chr(reg & 0xFF)
message = message.strip()
```

---

## Example — Citect / Elipse configuration

| Parameter | Value |
|---|---|
| Protocol | Modbus TCP |
| IP Address | IP of the machine running VMTA Gateway |
| Port | 502 (or as configured) |
| Unit ID | 1 (or as configured) |
| Poll interval | 1–5 seconds |

Map DI address 0 (`System`) as your primary watchdog — if it drops to `0` the gateway process has stopped.
