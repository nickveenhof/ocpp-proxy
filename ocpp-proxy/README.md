# OCPP Sniffer - Transparent OCPP Proxy for RFID Identification

A minimal, transparent OCPP 1.6 proxy that sits between your EV charger and your CPO backend (e.g. Wattify). It forwards all traffic unchanged and captures RFID idTags for use in EVCC vehicle identification.

## What it does

```
Wallbox ──wss──► OCPP Sniffer ──wss──► Wattify (CPO)
                     │
                     └──► /charger_info  (idTag for EVCC)
```

1. Wallbox connects to the proxy instead of directly to Wattify.
2. Every OCPP message is forwarded to Wattify unchanged.
3. Every Wattify response is relayed back to the charger unchanged.
4. Wattify remains in full control of authorization and billing.
5. The proxy sniffs `Authorize` and `StartTransaction` messages and records the `idTag`.
6. EVCC polls `/charger_info` to read the last `idTag` and identify the vehicle.

## Installation

### Home Assistant Add-on

1. In HA: **Settings > Add-ons > Add-on Store > Repositories**
2. Add: `https://github.com/nickveenhof/ocpp-proxy`
3. Install **OCPP Sniffer**
4. Configure (see below)
5. Start

## Configuration

```yaml
upstream_url: "wss://cpo.wattify.be/ocpp/YOUR_SERIAL"
api_token: "your-secret-token"
charger_password: "your-charger-password"
```

| Field | Required | Description |
|---|---|---|
| `upstream_url` | Yes | Your CPO OCPP endpoint (e.g. Wattify) |
| `charger_password` | Recommended | OCPP Basic Auth password. Set the same value in the Wallbox app OCPP password field. Rejects any charger connection without this password. |

## Charger setup

Point your charger's OCPP URL to the proxy instead of your CPO:

| Setting | Value |
|---|---|
| OCPP URL | `wss://ocpp.yourdomain.com/charger` |
| Identity | your charger serial number |
| Password | (empty) |

## REST API

All endpoints except `/charger` require `Authorization: Bearer <api_token>` when `api_token` is set.

### Read endpoints

| Endpoint | Description |
|---|---|
| `GET /charger_info` | idTag, status, vendor, firmware, serial |
| `GET /meter_values` | L1/L2/L3 voltage, current, power, energy (Wh) |
| `GET /last_session` | Last completed session: idTag, energy, duration, stop reason |
| `GET /data_transfer` | Last 20 vendor DataTransfer messages |
| `GET /status` | Upstream URL and connection state |
| `GET /sessions` | All completed sessions (JSON) |
| `GET /sessions.csv` | All completed sessions (CSV) |

### Command endpoints

| Endpoint | Description |
|---|---|
| `POST /enable/true` | RemoteStartTransaction |
| `POST /enable/false` | RemoteStopTransaction |
| `POST /maxcurrent/{amps}` | SetChargingProfile (e.g. `/maxcurrent/16`) |
| `POST /command` | Raw OCPP: `{"action":"ChangeAvailability","payload":{...}}` |

### `/charger_info` example

```json
{
  "connected": true,
  "vendor": "Wall Box Chargers",
  "model": "PPR1-0-2-4",
  "last_id_tag": "97BA7F51",
  "last_status": "Preparing",
  "firmware": "6.11.16",
  "serial": "1305884"
}
```

### `/meter_values` example

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

## EVCC integration

In `evcc.yaml`, use `type: custom` and poll `/charger_info` for the idTag:

```yaml
chargers:
  - name: wallbox
    type: custom
    status:
      source: homeassistant
      entity: sensor.wallbox_evcc_status
    enabled:
      source: homeassistant
      entity: sensor.wallbox_evcc_enabled
    enable:
      source: homeassistant
      entity: switch.wallbox_pulsar_pro_sn_1305884_pause_resume
    maxcurrent:
      source: homeassistant
      entity: number.wallbox_pulsar_pro_sn_1305884_maximum_charging_current
    power:
      source: homeassistant
      entity: sensor.wallbox_charge_power_w
    energy:
      source: homeassistant
      entity: sensor.wallbox_pulsar_pro_sn_1305884_added_energy
    identify:
      source: http
      uri: http://192.168.1.126:9000/charger_info
      jq: .last_id_tag

vehicles:
  - name: polestar4
    type: polestar
    identifiers:
      - 97BA7F51
```

## Architecture

This proxy does NOT act as a Central System. Wattify remains the Central System and handles all authorization and billing. The proxy is transparent to both sides.

The only thing the proxy does beyond forwarding is:
- Log `idTag` from `Authorize` and `StartTransaction` messages
- Expose `idTag` via REST for EVCC vehicle identification

## License

MIT
