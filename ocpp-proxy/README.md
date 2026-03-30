# OCPP Sniffer

A transparent OCPP 1.6 proxy for Home Assistant that sits between your EV charger
and your CPO (Charge Point Operator). It forwards all traffic unchanged while
capturing RFID idTags and meter values for use in evcc.

## Why does this exist?

If you use **evcc** for solar charging and your charger is managed by a CPO like
Wattify for billing, you face a problem:

- The CPO controls authorization via OCPP. Your charger only talks to the CPO.
- evcc needs to know **who plugged in** (RFID tag) to select the right vehicle and
  charging schedule.
- The CPO never exposes this information to evcc.

OCPP Sniffer solves this by sitting transparently in the middle.

## How it works

```
Without sniffer:

  Wallbox ──wss──► Wattify (CPO)
                      │
                      └── billing, auth, scheduling

  evcc ──HA API──► Wallbox  (no RFID, no meter data)


With sniffer:

  Wallbox ──wss──► OCPP Sniffer ──wss──► Wattify (CPO)
                        │                    │
                        │                    └── billing, auth, scheduling
                        │
                        ├── /charger_info   → RFID tag, charger status
                        ├── /meter_values   → power, energy, L1/L2/L3
                        └── /last_session   → session data
                        
  evcc ──HTTP──► OCPP Sniffer  (RFID tag, meter data, direct control)
```

Every OCPP message is forwarded to Wattify **unchanged**. Wattify remains in full
control of authorization and billing. The sniffer only reads and exposes data.

## What the sniffer captures

| OCPP message | What is captured | Where |
|---|---|---|
| `BootNotification` | Vendor, model, firmware, serial | `/charger_info` |
| `StatusNotification` | Charger status (A/B/C for evcc) | `/charger_info` |
| `Authorize` | RFID idTag | `/charger_info` |
| `StartTransaction` | RFID idTag, meter start | `/charger_info`, `/last_session` |
| `StopTransaction` | Meter stop, energy used, stop reason | `/last_session` |
| `MeterValues` | L1/L2/L3 voltage, current, power, energy | `/meter_values` |
| `DataTransfer` | Vendor-specific messages (last 20) | `/data_transfer` |

## Installation

1. In HA: **Settings > Add-ons > Add-on Store > ⋮ > Repositories**
2. Add: `https://github.com/nickveenhof/ocpp-sniffer`
3. Install **OCPP Sniffer**
4. Configure (see below)
5. Start

## Configuration

```yaml
upstream_url: "wss://cpo.wattify.be/ocpp/YOUR_SERIAL"
charger_password: "your-password"
```

| Field | Required | Description |
|---|---|---|
| `upstream_url` | Yes | Your CPO OCPP WebSocket URL |
| `charger_password` | Recommended | OCPP Basic Auth password. Set the same value in the Wallbox app OCPP password field. Rejects unauthenticated connections. |

## Charger setup

Change your charger's OCPP URL to point to the sniffer instead of your CPO.

| Setting | Old value | New value |
|---|---|---|
| OCPP URL | `wss://cpo.wattify.be/ocpp/1305884` | `wss://ocpp.yourdomain.com/charger` |
| Identity | `1305884` | `1305884` (unchanged) |
| Password | (empty) | your `charger_password` |

The sniffer is publicly reachable via a Cloudflare Tunnel (see below).

## Making the sniffer reachable

The charger needs a public WSS endpoint. Use a Cloudflare Tunnel:

1. Create a tunnel at [one.dash.cloudflare.com](https://one.dash.cloudflare.com)
2. Install the Cloudflare add-on in HA
3. Set `tunnel_token` in the Cloudflare add-on config
4. Add public hostname: `ocpp.yourdomain.com` → `http://172.30.33.X:9000`
   (find the sniffer container IP with `docker inspect addon_..._ocpp-proxy`)

## REST API

All endpoints are available at `http://SNIFFER_IP:9000` (LAN only, no auth needed
since evcc polls from within your network).

### Read endpoints

| Endpoint | Description |
|---|---|
| `GET /charger_info` | Charger state, RFID idTag, vendor info |
| `GET /meter_values` | L1/L2/L3 voltage, current, power, energy |
| `GET /last_session` | Last completed session: idTag, energy, duration, stop reason |
| `GET /data_transfer` | Last 20 vendor DataTransfer messages |
| `GET /status` | Upstream URL and connection state |
| `GET /sessions` | All completed sessions (JSON) |
| `GET /sessions.csv` | All completed sessions (CSV) |

### Command endpoints

These inject OCPP commands directly into the charger connection. Faster than the
Wallbox cloud API (no round trip to the cloud).

| Endpoint | OCPP command | Description |
|---|---|---|
| `POST /enable/true` | `RemoteStartTransaction` | Start charging |
| `POST /enable/false` | `RemoteStopTransaction` | Stop charging |
| `POST /maxcurrent/{amps}` | `SetChargingProfile` | Set max current (e.g. `/maxcurrent/16`) |
| `POST /command` | any | Raw OCPP: `{"action":"ChangeAvailability","payload":{...}}` |

### Example responses

`GET /charger_info`
```json
{
  "connected": true,
  "vendor": "Wall Box Chargers",
  "model": "PPR1-0-2-4",
  "firmware": "6.11.16",
  "serial": "1305884",
  "last_id_tag": "97BA7F51",
  "last_status": "Charging",
  "evcc_status": "C"
}
```

`GET /meter_values`
```json
{
  "energy_wh": 111335.0,
  "power_w": 7400.0,
  "current_l1": 10.5,
  "current_l2": 10.5,
  "current_l3": 10.4,
  "voltage_l1": 235.0,
  "voltage_l2": 229.0,
  "voltage_l3": 230.0,
  "timestamp": "2026-03-30T14:00:00Z"
}
```

`GET /last_session`
```json
{
  "id_tag": "97BA7F51",
  "transaction_id": 42,
  "start_time": "2026-03-30T08:00:00Z",
  "stop_time": "2026-03-30T10:30:00Z",
  "meter_start_wh": 111000.0,
  "meter_stop_wh": 118400.0,
  "energy_wh": 7400.0,
  "stop_reason": "Local"
}
```

## evcc configuration

Replace your existing `homeassistant` charger with this `custom` charger config.
All data comes directly from the sniffer via HTTP. No HA API calls needed.

```yaml
chargers:
  - name: wallbox
    type: custom

    # Charger status: A=not connected, B=connected, C=charging
    status:
      source: http
      uri: http://172.30.33.X:9000/charger_info
      jq: .evcc_status

    # Is charging enabled?
    enabled:
      source: http
      uri: http://172.30.33.X:9000/charger_info
      jq: .last_status != "Unavailable"

    # Enable/disable charging via direct OCPP command
    enable:
      source: http
      uri: http://172.30.33.X:9000/enable/{{.enable}}
      method: POST

    # Set max current via direct OCPP SetChargingProfile
    maxcurrent:
      source: http
      uri: http://172.30.33.X:9000/maxcurrent/{{.maxcurrent}}
      method: POST

    # Power in W from OCPP MeterValues
    power:
      source: http
      uri: http://172.30.33.X:9000/meter_values
      jq: .power_w

    # Energy in kWh from OCPP MeterValues
    energy:
      source: http
      uri: http://172.30.33.X:9000/meter_values
      jq: .energy_wh / 1000

    # RFID tag for vehicle identification
    identify:
      source: http
      uri: http://172.30.33.X:9000/charger_info
      jq: .last_id_tag

    # Phase currents in A
    currents:
      - source: http
        uri: http://172.30.33.X:9000/meter_values
        jq: .current_l1
      - source: http
        uri: http://172.30.33.X:9000/meter_values
        jq: .current_l2
      - source: http
        uri: http://172.30.33.X:9000/meter_values
        jq: .current_l3

    # Phase voltages in V
    voltages:
      - source: http
        uri: http://172.30.33.X:9000/meter_values
        jq: .voltage_l1
      - source: http
        uri: http://172.30.33.X:9000/meter_values
        jq: .voltage_l2
      - source: http
        uri: http://172.30.33.X:9000/meter_values
        jq: .voltage_l3
```

Replace `172.30.33.X` with your sniffer container IP.

## Vehicle identification

Add the RFID idTag to each vehicle's `identifiers` list in evcc:

```yaml
vehicles:
  - name: polestar4
    type: polestar
    # ... your existing config ...
    identifiers:
      - 97BA7F51   # your RFID tag from /charger_info
```

evcc matches the `identify` value against vehicle identifiers at session start and
automatically selects the right vehicle and charging schedule.

### Finding your RFID tag

1. Point your charger at the sniffer.
2. Plug in your car (not just tap RFID — the tag appears in `StartTransaction`).
3. Check `GET /charger_info` → `.last_id_tag`.

## Important notes

**One upstream only.** The sniffer forwards to exactly one CPO. Multiple backends
are not supported and not planned.

**Wattify controls auth.** If your CPO uses a local authorization list
(`SendLocalList`), the charger may authorize locally without sending `Authorize`
over the wire. The RFID tag still appears in `StartTransaction` when the car is
plugged in.

**MeterValues appear only during charging.** The `/meter_values` endpoint returns
zeros until a charging session starts and the charger sends `MeterValues`.

**BootNotification.** Vendor, model, firmware and serial are captured from
`BootNotification`. This only fires on full power cycle of the charger. Soft
reconnects skip it.

## License

MIT
