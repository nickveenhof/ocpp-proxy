# Cloudflare Tunnel Setup

Your charger connects over WSS and needs a valid TLS certificate. A Cloudflare Tunnel provides this for free.

```
Charger ──wss──► ocpp.yourdomain.com  (Cloudflare edge, valid TLS)
                        │
                   Cloudflare Tunnel
                        │
                   HA host (LAN) :9000
```

## Step 1: Add your domain to Cloudflare

Free plan works. Transfer nameservers from your registrar to Cloudflare. If DNSSEC is enabled: disable it at your registrar first, change nameservers, re-enable DNSSEC in Cloudflare after activation.

## Step 2: Create a tunnel

1. [one.dash.cloudflare.com](https://one.dash.cloudflare.com) > **Networks > Tunnels > Create**
2. Connector: **Cloudflared**. Name: anything.
3. Copy the tunnel token (`eyJ...`).

## Step 3: Install Cloudflared add-on in HA

1. HA: **Settings > Add-ons > Add-on Store > ⋮ > Repositories**
2. Add: `https://github.com/homeassistant-apps/repository`
3. Install **Cloudflared**. Set `tunnel_token` in config.
4. Start. Dashboard shows **Connected**.

## Step 4: Add public hostname

Cloudflare Zero Trust > **Tunnels > your tunnel > Public Hostname > Add**:

| Field | Value |
|---|---|
| Subdomain | `ocpp` |
| Domain | `yourdomain.com` |
| Type | `HTTP` |
| URL | `SNIFFER_CONTAINER_IP:9000` |

Find the container IP:
```bash
docker inspect addon_XXXXX_ocpp-proxy \
  --format '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}'
```

## Done

Your charger can now connect to `wss://ocpp.yourdomain.com/charger`.
