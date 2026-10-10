# ThingsBoard on Dokploy

Dashboard and MQTT are both reached through Traefik with TLS termination.
No service publishes a host port directly - Postgres, Kafka, and ThingsBoard
are only reachable over the internal `dokploy-network`.

## 1. Set up environment variables

Copy `.env.example` to `.env` and fill in real values (or set the same keys
as "Environment Variables" in the Dokploy app UI):

- `DASHBOARD_DOMAIN` - hostname for the ThingsBoard web UI (e.g. `tb.yourdomain.com`)
- `MQTT_DOMAIN` - hostname used for MQTTS SNI routing (e.g. `mqtt.yourdomain.com`)
- `TRAEFIK_CERT_RESOLVER` - the cert resolver name configured in Dokploy's Traefik (default `letsencrypt`)
- `POSTGRES_DB`, `POSTGRES_USER`, `POSTGRES_PASSWORD` - database settings and secrets
- `LOAD_DEMO`, `JAVA_OPTS`, `SECURITY_CORS_ALLOWED_ORIGINS` - ThingsBoard runtime options (see `.env.example`)

Both `DASHBOARD_DOMAIN` and `MQTT_DOMAIN` need DNS A/AAAA records pointing at
your Dokploy server.

## 2. One-time Traefik change in Dokploy (for MQTTS)

Traefik routes HTTP by `Host` header, but MQTT has no such header. Traefik
can still terminate TLS for MQTT using a **TCP router with TLS + SNI
routing** (`HostSNI`), but only for the *encrypted* MQTT port (8883/MQTTS) -
a client's TLS ClientHello carries the SNI hostname Traefik reads to route
the connection, then Traefik decrypts and forwards plain MQTT to
ThingsBoard on 1883 internally. Plain unencrypted MQTT (1883) has no TLS
layer, so it cannot be routed or exposed this way, and is intentionally not
published - only MQTTS (8883) is reachable from outside.

Dokploy's Traefik only listens on 80/443 by default, so you must add a new
entrypoint once:

1. In Dokploy, go to your server's **Traefik** settings (Advanced /
   Traefik configuration section).
2. Add a new static entrypoint, e.g.:
   ```yaml
   entryPoints:
     mqtts:
       address: ":8883"
   ```
3. Make sure the Traefik container itself publishes port `8883` to the
   host (Dokploy's server/Traefik advanced settings usually let you add
   extra port mappings for the Traefik service).
4. Redeploy Traefik so the new entrypoint takes effect.

After that, this compose file's labels (`entrypoints=mqtts`) will pick up
traffic on 8883 automatically - no further Traefik config needed per
deploy.

## 3. Deploy

Deploy this docker-compose stack as a Compose application in Dokploy,
pointing it at the `dokploy-network` (already external in this file).
ThingsBoard's dashboard will be available at `https://<DASHBOARD_DOMAIN>`
and MQTTS at `<MQTT_DOMAIN>:8883` with a valid TLS cert issued by Traefik.

## What changed from the original compose

- Removed all `ports:` publishing (Postgres 5432, Kafka 9092/9093,
  ThingsBoard 8080/7070/1883/8883/5683-5688). Nothing is exposed directly
  anymore; only Traefik has a path in.
- Added Traefik labels on `thingsboard-ce` for:
  - HTTPS routing to the dashboard (port 8080), with HTTP -> HTTPS redirect.
  - MQTTS TCP routing (port 1883 internally) via SNI + TLS termination at
    Traefik.
- CoAP (5683-5688/udp) and the 7070 RPC port were dropped since they
  weren't requested and Traefik can't TLS-terminate arbitrary UDP/DTLS
  traffic anyway. Say the word if you need those reachable too.
- Secrets (`POSTGRES_PASSWORD`, `POSTGRES_DB`) and domains now come from
  `.env` instead of being hardcoded.
- Added a private `tb-internal` network (`internal: true`, no external/
  internet access) for Postgres <-> Kafka <-> ThingsBoard traffic. Only
  `thingsboard-ce` also joins `dokploy-network`, so Postgres and Kafka are
  unreachable from Traefik or any other app Dokploy manages on the shared
  network.
- Init work uses Compose [`pre_start`](https://docs.docker.com/reference/compose-file/services/#pre_start)
  steps instead of one-shot init services. They run on `compose up` /
  recreate only, so after a Dokploy or host restart the containers simply
  start (`restart: always`) without waiting for jobs that never re-run.
  Requires a Docker Compose version that supports `pre_start`.
  - `postgres`, `kafka` and `thingsboard-ce` each chown their own volumes
    to their fixed UID:GID (root-owned on first creation).
  - `thingsboard-ce` then runs `INSTALL_TB=true` against Postgres exactly
    once. It writes a marker file (`/data/.thingsboard-installed`, on the
    `thingsboard-install` volume, mounted into `thingsboard-ce` for this
    purpose) so re-deploys never re-run the installer against an
    already-initialized database. Set `LOAD_DEMO=true` in `.env` before the
    *first* deploy if you want demo data seeded - it has no effect after
    that first run.
