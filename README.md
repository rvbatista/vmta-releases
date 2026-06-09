# VMTA Gateway

VMTA Gateway emulates a GSM modem (MultiTech MTD-H5 / MTC-H5-B03) over a virtual COM port pair, allowing legacy SCADA software to send SMS and email through modern cloud APIs — no hardware modem required.

```
Legacy SCADA (COM5) ◄── com0com ──► VMTA Gateway (COM6)
                                          │
                              ┌───────────┼───────────┐
                              ▼           ▼           ▼
                        Twilio SMS   Telegram    Email (SendGrid / SMTP)
```

It also includes a SCADA alarm engine that tails an alarm file or serial port and dispatches filtered alarms directly to phone numbers, Telegram chats, and email addresses — with windowing, cooldowns, throttling, and retry queuing.

---

## Download

Go to the [Releases](../../releases) page and download the binary for your platform:

| File | Platform |
|---|---|
| `vmtagateway.exe` | Windows 10 / 11 |
| `vmtagateway` | Linux (x86-64) |

The binary is fully standalone — no Python or dependencies required.

---

## First-Run Setup

### 1. Place the binary in a dedicated folder

```
C:\vmtagateway\
    vmtagateway.exe
```

On first run the gateway creates default configuration files and exits:

```
vmtagateway.exe
```

You will see:
```
Default configuration files were created.
Please edit config.yaml and .env with your settings, then restart.
```

### 2. Edit `.env`

Fill in your API credentials:

```env
# Twilio (SMS)
TWILIO_ACCOUNT_SID=ACxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
TWILIO_AUTH_TOKEN=your_auth_token
TWILIO_FROM_NUMBER=+1234567890
TWILIO_TO_NUMBER=+0987654321

# Telegram
TELEGRAM_BOT_TOKEN=123456789:ABCdef...
TELEGRAM_CHAT_ID=123456789

# Email — SendGrid
SENDGRID_API_KEY=SG.xxxxxxxxxxxx
SENDGRID_FROM=gateway@yourdomain.com
SENDGRID_TO=destination@yourdomain.com

# License
VMTA_LICENSE_KEY=VMTA-xxxxxxxxxxxx
```

### 3. Edit `config.yaml`

Key settings:

| Setting | Description |
|---|---|
| `serial.port` | COM port the gateway opens (must match `com0com.gateway_port`) |
| `com0com.client_port` | Port your SCADA software connects to (e.g. `COM5`) |
| `com0com.gateway_port` | Port this gateway opens (e.g. `COM6`) |
| `channels.twilio_sms.enabled` | Enable Twilio SMS forwarding |
| `channels.telegram.enabled` | Enable Telegram forwarding |
| `smtp.enabled` | Enable SMTP gateway (email forwarding) |
| `scada.enabled` | Enable SCADA alarm engine |
| `scada.source.file_path` | Path to your SCADA alarm TSV/CSV file |
| `system_id` | Label shown in all notifications (e.g. `PLANT-A`) |

### 4. Edit `destinations.yaml`

Define contact groups — the people who receive alarms:

```yaml
scada:
  contact_groups:
    maintenance:
      twilio_sms: ["+5511999990000"]
      telegram_chats: ["123456789"]
      emails: ["oncall@yourcompany.com"]
```

### 5. Edit `rules.yaml`

Define which alarms go where:

```yaml
scada:
  rules:
    - name: "CRITICAL"
      match: "(?i)TRIP|FAULT|FIRE"
      destinations: ["maintenance"]
      window_seconds: 10
      cooldown_seconds: 300
```

> `destinations.yaml` and `rules.yaml` are hot-reloaded every 30 seconds — no restart needed when updating rules or contacts.

---

## Running

### Foreground (test / dry-run)

**Windows:**
```
vmtagateway.exe --dry-run
```

**Linux:**
```bash
chmod +x vmtagateway
./vmtagateway --dry-run
```

`--dry-run` prints all messages to the console without making real API calls.

### Windows Service

```
vmtagateway.exe install
vmtagateway.exe start
```

To stop or remove:
```
vmtagateway.exe stop
vmtagateway.exe remove
```

### Linux systemd

```bash
./vmtagateway install
sudo cp vmtagateway.service /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable --now vmtagateway
```

---

## com0com (Windows only)

Download and install [com0com](https://com0com.sourceforge.net/). The gateway manages the virtual COM port pair automatically on startup. Run as Administrator the first time.

---

## Automatic Updates (OTA)

Enable in `config.yaml` to have the gateway check for new releases automatically:

```yaml
update:
  enabled: true
  github_repo: "rvbatista/vmta-releases"
  check_interval: 86400   # seconds (default: 24 hours)
```

When a new release is available the gateway downloads it, backs up the current binary, replaces it, and restarts the service automatically. If the new binary fails to start, it rolls back to the backup.

---

## Modbus Diagnostic Server

The gateway exposes a Modbus TCP server (default port 502) for health monitoring from SCADA systems (Citect, Elipse, etc.):

```yaml
diagnostics:
  modbus:
    enabled: true
    port: 502
    unit_id: 1
```

| Address | Type | Description |
|---|---|---|
| 0–10 | Discrete Inputs | Component health bits (system, license, modem, SMTP, SCADA…) |
| 0–25 | Holding Registers | Stats counters, uptime, last alarm timestamp |

---

## License

A valid license key (`VMTA_LICENSE_KEY` in `.env`) is required for production use. Without it the gateway runs in demo mode: messages are redacted and the process stops after 24 hours.

To get your machine's hardware ID for licensing:

```
vmtagateway.exe --show-id
```
