# OCPP Sniffer

Transparent OCPP 1.6 proxy for Home Assistant. Sits between your charger and your CPO. Forwards all traffic unchanged. Captures RFID tags and meter data for evcc.

## The problem

Your charger is managed by a CPO for billing via OCPP. evcc cannot see who plugged in, so it cannot select the right vehicle or schedule.

## The solution

```
Charger ──wss──► OCPP Sniffer ──wss──► CPO  (billing unchanged)
                      │
                      ├── /charger_info   RFID tag, status
                      ├── /meter_values   power, energy, L1/L2/L3
                      ├── /enable/{bool}  pause/resume (SetChargingProfile)
                      └── /maxcurrent/N   set current (SetChargingProfile)
evcc ──HTTP──► OCPP Sniffer
```

## Install

1. HA: **Settings > Add-ons > Add-on Store > ⋮ > Repositories**
2. Add: `https://github.com/nickveenhof/ocpp-sniffer`
3. Install **OCPP Sniffer**.

## Config

In the add-on Configuration tab, fill in:

| Field | What to enter |
|---|---|
| upstream_url | Your CPO OCPP URL, e.g. `wss://cpo.example.com/ocpp/123456` |
| charger_password | A password. Set the same in your charger's OCPP settings. |
| min_current | Minimum charge current in amps. Default: `6`. |
| auto_throttle | `true` (default). Sets 0A on plug-in. evcc controls when to start. |

## Auto-throttle

When enabled, the sniffer sets 0A immediately after `StartTransaction`. The CPO session starts (billing runs), but the charger draws no power. evcc decides when to charge via `/enable/true`.

Without it, the charger starts at full power on plug-in. evcc reacts after 10-30 seconds.

## Charger setup

Your charger needs to reach the sniffer over WSS with a valid TLS cert. Use a Cloudflare Tunnel. See [CLOUDFLARE.md](CLOUDFLARE.md) for setup instructions.

Then in your charger's OCPP settings:

| Setting | Value |
|---|---|
| OCPP URL | `wss://ocpp.yourdomain.com/charger` |
| Identity | your charger serial number |
| Password | same as `charger_password` above |

## What gets captured

| OCPP message | Data | Endpoint |
|---|---|---|
| `BootNotification` | Vendor, model, firmware, serial | `/charger_info` |
| `StatusNotification` | Status (A/B/C for evcc) | `/charger_info` |
| `Authorize` | RFID idTag | `/charger_info` |
| `StartTransaction` | RFID idTag, meter start | `/charger_info`, `/last_session` |
| `StopTransaction` | Meter stop, energy, stop reason | `/last_session` |
| `MeterValues` | L1/L2/L3 voltage, current, power, energy | `/meter_values` |
| `DataTransfer` | Vendor messages (last 20) | `/data_transfer` |

## REST API

### Read

| Endpoint | Returns |
|---|---|
| `GET /charger_info` | `{"connected", "evcc_status", "last_id_tag", "last_status", "vendor", "model", "firmware", "serial"}` |
| `GET /meter_values` | `{"power_w", "energy_wh", "current_l1/l2/l3", "voltage_l1/l2/l3", "timestamp"}` |
| `GET /last_session` | `{"id_tag", "transaction_id", "start_time", "stop_time", "energy_wh", "stop_reason"}` |
| `GET /status` | `{"charger_connected", "upstream"}` |
| `GET /data_transfer` | Last 20 vendor DataTransfer messages |
| `GET /sessions` | All completed sessions (JSON) |
| `GET /sessions.csv` | All completed sessions (CSV) |

### Commands

| Endpoint | Effect |
|---|---|
| `POST /enable/true` | Resume charging at `min_current` amps |
| `POST /enable/false` | Pause charging (0A) |
| `POST /maxcurrent/{amps}` | Set max current |
| `POST /command` | Raw OCPP: `{"action":"...","payload":{...}}` |

## evcc custom charger config

Paste this in the evcc UI custom charger YAML field. Replace `SNIFFER_IP` with your container IP.

```yaml
status:
  source: http
  uri: http://SNIFFER_IP:9000/charger_info
  jq: .evcc_status
enabled:
  source: http
  uri: http://SNIFFER_IP:9000/charger_info
  jq: .last_status != "Unavailable"
enable:
  source: http
  uri: http://SNIFFER_IP:9000/enable/{{.enable}}
  method: POST
maxcurrent:
  source: http
  uri: http://SNIFFER_IP:9000/maxcurrent/{{.maxcurrent}}
  method: POST
power:
  source: http
  uri: http://SNIFFER_IP:9000/meter_values
  jq: .power_w
energy:
  source: http
  uri: http://SNIFFER_IP:9000/meter_values
  jq: .energy_wh / 1000
identify:
  source: http
  uri: http://SNIFFER_IP:9000/charger_info
  jq: .last_id_tag
currents:
  - source: http
    uri: http://SNIFFER_IP:9000/meter_values
    jq: .current_l1
  - source: http
    uri: http://SNIFFER_IP:9000/meter_values
    jq: .current_l2
  - source: http
    uri: http://SNIFFER_IP:9000/meter_values
    jq: .current_l3
voltages:
  - source: http
    uri: http://SNIFFER_IP:9000/meter_values
    jq: .voltage_l1
  - source: http
    uri: http://SNIFFER_IP:9000/meter_values
    jq: .voltage_l2
  - source: http
    uri: http://SNIFFER_IP:9000/meter_values
    jq: .voltage_l3
```

## Vehicle identification

Add your RFID tag to the vehicle's identifiers in evcc. Find your tag by plugging in and checking `GET /charger_info` → `last_id_tag`.

## Notes

**One upstream only.** No multi-backend support.

**Local auth.** CPOs using `SendLocalList` authorize locally. The idTag appears in `StartTransaction` at plug-in, not at RFID tap.

**MeterValues.** Returns zeros until a charging session starts.

**BootNotification.** Vendor/model/firmware populate on full power cycle only.

**Tested with.** Wallbox Pulsar Pro + Wattify CPO.

## License

MIT
