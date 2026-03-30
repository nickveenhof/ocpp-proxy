# OCPP Sniffer

Transparent OCPP 1.6 proxy for Home Assistant. Sits between your charger and your CPO. Forwards everything unchanged. Captures RFID tags and meter data for evcc.

## The problem

You use evcc for solar charging. Your charger is managed by a CPO (e.g. Wattify) for billing. The CPO controls who can charge via OCPP. evcc has no idea who plugged in, so it cannot select the right vehicle or schedule.

## The solution

```
Without sniffer:

  Charger ──wss──► CPO  (billing, auth, scheduling)
  evcc ──HA API──► Charger  (no RFID, no meter data)

With sniffer:

  Charger ──wss──► OCPP Sniffer ──wss──► CPO  (billing unchanged)
                        │
                        ├── /charger_info   RFID tag, charger status
                        ├── /meter_values   power, energy, L1/L2/L3
                        └── /last_session   session data
  evcc ──HTTP──► OCPP Sniffer
```

The CPO stays in full control. The sniffer only reads.

## What gets captured

| OCPP message | Captured | Endpoint |
|---|---|---|
| `BootNotification` | Vendor, model, firmware, serial | `/charger_info` |
| `StatusNotification` | Status (A/B/C for evcc) | `/charger_info` |
| `Authorize` | RFID idTag | `/charger_info` |
| `StartTransaction` | RFID idTag, meter start | `/charger_info`, `/last_session` |
| `StopTransaction` | Meter stop, energy used, stop reason | `/last_session` |
| `MeterValues` | L1/L2/L3 voltage, current, power, energy | `/meter_values` |
| `DataTransfer` | Vendor messages (last 20) | `/data_transfer` |

## Install

1. HA: **Settings > Add-ons > Add-on Store > ⋮ > Repositories**
2. Add: `https://github.com/nickveenhof/ocpp-sniffer`
3. Install **OCPP Sniffer**, configure, start.

## Config

```yaml
upstream_url: "wss://your-cpo-endpoint/ocpp/YOUR_CHARGER_ID"
charger_password: "choose-a-strong-password"
```

| Field | Required | Description |
|---|---|---|
| `upstream_url` | Yes | Your CPO OCPP WebSocket URL |
| `charger_password` | Recommended | OCPP Basic Auth password. Set the same value in your charger's OCPP password field. Rejects unauthenticated connections. |

## Making the sniffer reachable

Your charger connects over WSS and needs a valid TLS certificate. A local IP does not work. Use a **Cloudflare Tunnel**.

```
Charger ──wss──► ocpp.yourdomain.com  (Cloudflare edge, valid TLS)
                        │
                   Cloudflare Tunnel
                        │
                   HA host (LAN) :9000
```

### Step 1: Add your domain to Cloudflare

Free plan works. Either transfer nameservers or register a new domain.

If your domain is at another registrar: disable DNSSEC there first, change nameservers to Cloudflare's, re-enable DNSSEC in Cloudflare after activation.

### Step 2: Create a tunnel

1. [one.dash.cloudflare.com](https://one.dash.cloudflare.com) > **Networks > Tunnels > Create**
2. Connector: **Cloudflared**. Name: anything. Click **Save**.
3. Copy the tunnel token (`eyJ...`).

### Step 3: Install Cloudflared add-on in HA

1. HA: **Settings > Add-ons > Add-on Store > ⋮ > Repositories**
2. Add: `https://github.com/homeassistant-apps/repository`
3. Install **Cloudflared**. Set config:
   ```yaml
   tunnel_token: "eyJ...your token..."
   ```
4. Start. Cloudflare dashboard shows tunnel as **Connected**.

### Step 4: Add public hostname

Cloudflare Zero Trust > **Tunnels > your tunnel > Configure > Public Hostname > Add**:

| Field | Value |
|---|---|
| Subdomain | `ocpp` |
| Domain | `yourdomain.com` |
| Type | `HTTP` |
| URL | `172.30.33.X:9000` |

Find the sniffer container IP:
```bash
docker inspect addon_XXXXX_ocpp-proxy \
  --format '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}'
```

### Step 5: Point your charger at the sniffer

| Setting | Value |
|---|---|
| OCPP URL | `wss://ocpp.yourdomain.com/charger` |
| Identity | your charger serial number |
| Password | your `charger_password` |

## REST API

Available at `http://SNIFFER_IP:9000`. LAN only. No auth needed for read endpoints.

### Read

| Endpoint | Description |
|---|---|
| `GET /` | Web UI with all endpoints |
| `GET /charger_info` | RFID tag, status, vendor, firmware |
| `GET /meter_values` | L1/L2/L3 voltage, current, power, energy |
| `GET /last_session` | Last session: idTag, energy, duration, stop reason |
| `GET /data_transfer` | Last 20 vendor DataTransfer messages |
| `GET /status` | Upstream URL, connection state |
| `GET /sessions` | All sessions (JSON) |
| `GET /sessions.csv` | All sessions (CSV) |

### Commands (direct OCPP, no cloud round-trip)

| Endpoint | OCPP command | Description |
|---|---|---|
| `POST /enable/true` | `RemoteStartTransaction` | Start charging |
| `POST /enable/false` | `RemoteStopTransaction` | Stop charging |
| `POST /maxcurrent/{amps}` | `SetChargingProfile` | Set max current |
| `POST /command` | any | `{"action":"...","payload":{...}}` |

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

## evcc config

Replace your `homeassistant` charger with this. No HA API needed.

```yaml
chargers:
  - name: wallbox
    type: custom

    status:
      source: http
      uri: http://172.30.33.X:9000/charger_info
      jq: .evcc_status

    enabled:
      source: http
      uri: http://172.30.33.X:9000/charger_info
      jq: .last_status != "Unavailable"

    enable:
      source: http
      uri: http://172.30.33.X:9000/enable/{{.enable}}
      method: POST

    maxcurrent:
      source: http
      uri: http://172.30.33.X:9000/maxcurrent/{{.maxcurrent}}
      method: POST

    power:
      source: http
      uri: http://172.30.33.X:9000/meter_values
      jq: .power_w

    energy:
      source: http
      uri: http://172.30.33.X:9000/meter_values
      jq: .energy_wh / 1000

    identify:
      source: http
      uri: http://172.30.33.X:9000/charger_info
      jq: .last_id_tag

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

Add the RFID tag to each vehicle's `identifiers` in evcc:

```yaml
vehicles:
  - name: polestar4
    type: polestar
    identifiers:
      - 97BA7F51
```

evcc matches `identify` against vehicle identifiers at session start. Correct vehicle and schedule selected automatically.

### Finding your RFID tag

1. Point charger at sniffer.
2. Plug in your car (tag appears in `StartTransaction`, not just RFID tap).
3. `GET /charger_info` → `.last_id_tag`.

## Notes

**One upstream only.** One CPO. No multi-backend support. Not planned.

**Local auth.** CPOs using `SendLocalList` authorize locally. No `Authorize` message on the wire. The idTag still appears in `StartTransaction` at plug-in.

**MeterValues.** `/meter_values` returns zeros until a charging session starts.

**BootNotification.** Vendor/model/firmware populate on full power cycle only. Soft reconnects skip it.

**Tested with.** Wallbox Pulsar Pro + Wattify CPO. Other OCPP 1.6 chargers and CPOs should work but are untested.

## License

MIT
