# Moonraker Configuration (`moonraker.cfg`)

Moonraker is the API server that sits between Klipper and web frontends like Mainsail. It handles file uploads, print job management, and integrations with external services.

**Config file:** `moonraker.cfg`
**Not included by ramps.cfg** -- Moonraker reads this file independently from its own service.

---

## File Manager

```ini
[file_manager]
enable_object_processing: False
queue_gcode_uploads: True
```

| Setting | Value | Meaning |
|---------|-------|---------|
| `enable_object_processing` | `False` | Disables Moonraker's G-code post-processing for object cancellation. When `True`, Moonraker parses uploaded G-code to identify individual objects so you can cancel them mid-print. Disabled here, possibly because it adds processing overhead or wasn't needed. See [Configuration Review](configuration-review.md) for a note on this. |
| `queue_gcode_uploads` | `True` | Uploaded files go into the print queue instead of starting immediately. This lets you upload multiple files and manage the print order. |

---

## Job Queue

```ini
[job_queue]
load_on_startup: True
automatic_transition: False
```

| Setting | Value | Meaning |
|---------|-------|---------|
| `load_on_startup` | `True` | Restores the job queue after a Moonraker restart. If you had queued prints, they'll still be there. |
| `automatic_transition` | `False` | After a print finishes, the next job in the queue does NOT start automatically. You must manually start it. This is safer -- you want to remove the finished print and check the bed before starting another. |

---

## Power Control (Home Assistant Integration)

```ini
[power printer_psu]
type: homeassistant
protocol: https
address: hass.local
port: 443
device: switch.3d_printer_psu
token: long-ass-token
domain: switch
initial_state: off
off_when_shutdown: True
on_when_job_queued: True
```

This integrates the printer's power supply with Home Assistant, allowing Moonraker to turn the printer on/off via a smart switch.

### Connection Settings

| Setting | Value | Meaning |
|---------|-------|---------|
| `type` | `homeassistant` | Uses the Home Assistant API for power control |
| `protocol` | `https` | Encrypted connection to Home Assistant |
| `address` | `hass.local` | Home Assistant hostname (resolved via mDNS on your local network) |
| `port` | `443` | Standard HTTPS port |
| `token` | `long-ass-token` | Home Assistant Long-Lived Access Token for API authentication. Generate this in Home Assistant under Profile > Long-Lived Access Tokens. **This is a secret -- keep it out of public repositories.** |

### Device Settings

| Setting | Value | Meaning |
|---------|-------|---------|
| `device` | `switch.3d_printer_psu` | The Home Assistant entity ID for the smart plug/relay controlling the printer's power supply |
| `domain` | `switch` | The Home Assistant domain (switch = a simple on/off device) |

### Automation Behavior

| Setting | Value | Meaning |
|---------|-------|---------|
| `initial_state` | `off` | When Moonraker starts, the PSU defaults to off. The printer won't power on until explicitly requested. |
| `off_when_shutdown` | `True` | When Klipper shuts down (or Moonraker stops), the PSU is automatically turned off. Saves power and is a safety measure. |
| `on_when_job_queued` | `True` | When a print job is added to the queue, the PSU is automatically turned on. This means you can upload a file to Mainsail and the printer will power itself on, ready to print. |

### Typical Workflow

1. Upload a G-code file to Mainsail
2. Moonraker queues the job and turns on the PSU automatically
3. The printer powers up, Klipper connects
4. You start the print from Mainsail
5. After the print finishes (or Klipper shuts down), the PSU turns off automatically

### Security Note

The Home Assistant token in this file grants API access to your Home Assistant instance. If you share this config publicly (e.g., on GitHub), replace the token with a placeholder or use Moonraker's secrets feature:

```ini
token: {secrets.home_assistant.token}
```

With a separate `moonraker_secrets.ini` file (not committed to git) containing:
```ini
[home_assistant]
token = your-actual-token-here
```
