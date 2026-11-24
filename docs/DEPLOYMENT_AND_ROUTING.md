# Fleetbase Deployment and Routing Guide

This guide records the Docker deployment troubleshooting completed for the Bwongo Digital Solutions Fleetbase checkout and describes how to configure Fleetbase routing, including a self-hosted Valhalla service for Uganda.

> **Environment note:** The local deployment described here was tested with the Console footer showing v0.7.75 and the API health endpoint reporting v0.8.2. Those component versions are not a confirmed compatibility matrix. Check the running image tags and extension versions before applying settings copied from newer Fleetbase releases.

## 1. Deployment summary

| Component | Local address |
|---|---|
| Fleetbase Console | http://localhost:4200 |
| Fleetbase API / HTTPD | http://localhost:8001 |
| Fleetbase SocketCluster | localhost:38000 |
| Portainer | https://localhost:9443 |
| MySQL | localhost:3306 |

Project directory used: `~/bwongo-digital-solutions/fleetbase`

Use Docker Compose v2 commands (`docker compose`).

## 2. Deployment troubleshooting record

### 2.1 Console image build: GitHub SSH host-key scan

The Console build initially failed at an SSH host-key scan for GitHub. DNS resolution worked, but `ssh-keyscan github.com` did not return a host key in the build environment. The Console dependency manifests were checked and did not require SSH-based GitHub dependencies.

The unnecessary SSH host-key scan was removed. Git remained installed in the image:

```dockerfile
RUN apk update && apk add --no-cache git
```

Afterward, the frozen pnpm install and Ember production build completed and the `fleetbase-console:latest` image was produced.

### 2.2 HTTPD port conflict

Fleetbase initially attempted to publish host port 8000, which was already allocated to Portainer. Portainer was left running.

The local working copy's HTTPD mapping was changed from:

```yaml
- "8000:80"
```

to:

```yaml
- "8001:80"
```

The container still listens on port 80; host access is through port 8001. The SocketCluster mapping `38000:8000` was not changed.

Verify the API:

```bash
curl -i http://localhost:8001/
```

Expected response is HTTP 200 with a Fleetbase API JSON message.

### 2.3 Console API host and CORS symptoms

The Console initially called `http://localhost:8000/int/v1/onboard/should-onboard`. Because that host port belonged to Portainer, the request returned 404 and the browser reported a missing CORS header. The root cause was the incorrect API destination, not a CORS policy that should be worked around.

In the local working copy, the API host was changed from `http://localhost:8000` to `http://localhost:8001` in:

- `console/environments/.env.development`
- `console/fleetbase.config.json`

The Console was rebuilt and recreated:

```bash
docker compose build --no-cache console
docker compose up -d --force-recreate console
```

Then the browser was hard-refreshed with Ctrl+Shift+R.

**Important source-control check:** when this guide was added, the GitHub `prod` branch's tracked `docker-compose.yml` and Console configuration still contained the original port 8000 values. The port 8001 changes above were observed in the working deployment but were not part of this documentation-only commit. Keep the production configuration and documentation in sync before recreating services from a fresh checkout.

### 2.4 First-time Fleetbase initialization

The Console initially displayed “Fleetbase is not installed or configured”. The deployment script was run inside the API container:

```bash
docker compose exec application bash -c "./deploy.sh"
```

The script attempted to create `fleetbase_sandbox`, but the MySQL account `fleetbase` did not have permission. The database was created and the account granted access using the MySQL root account inside the database container:

```sql
CREATE DATABASE IF NOT EXISTS fleetbase_sandbox
  CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
GRANT ALL PRIVILEGES ON fleetbase_sandbox.* TO 'fleetbase'@'%';
FLUSH PRIVILEGES;
```

The next run reported the same issue for `fleetbase_storefront`. It was created and granted similarly:

```sql
CREATE DATABASE IF NOT EXISTS fleetbase_storefront
  CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
GRANT ALL PRIVILEGES ON fleetbase_storefront.* TO 'fleetbase'@'%';
FLUSH PRIVILEGES;
```

The grants were verified with:

```sql
SHOW GRANTS FOR 'fleetbase'@'%';
```

The deployment then completed and the user confirmed Fleetbase started.

These grants are scoped to the named databases. Do not grant global privileges just to bypass a database-creation error. Do not run `docker compose down -v` as a troubleshooting shortcut; it removes persistent volumes.

### 2.5 Useful operational commands

```bash
cd ~/bwongo-digital-solutions/fleetbase
docker compose ps
docker compose logs --tail=100 application
docker compose logs --tail=100 scheduler
docker compose logs --tail=100 httpd
docker compose restart application queue scheduler httpd
curl -i http://localhost:8001/
```

A previous status snapshot showed the scheduler as unhealthy. Re-check `docker compose ps` and scheduler logs before considering all background services healthy.

## 3. How Fleetbase routing works

Fleetbase separates two settings:

- **Routing service (`display_engine`)**: calculates and draws the road route, distance and ETA.
- **Route optimization engine (`optimization_engine`)**: changes the order of stops before the route is rendered.

Valhalla can be used for routing and for stop optimization. VROOM is an optimization engine; it does not render the route map. The distance-unit setting controls whether the UI displays kilometres or miles; it does not change the underlying route calculation.

The settings are under **Fleet-Ops → Settings → Routing**. The list of available engines is registry/extension-driven.

## 4. Valhalla: choose hosted or self-hosted

There are two distinct ways to use Valhalla:

1. **Fleetbase Hosted Routing**: use a Fleetbase-managed Valhalla endpoint and the key/plan offered by Fleetbase. This avoids maintaining map tiles and the routing server, but route requests are handled by an external provider. Review its current terms, data handling and usage limits before sending real customer locations.
2. **Self-hosted Valhalla**: run the Valhalla engine on infrastructure you control, build its routing graph from OpenStreetMap data, and configure Fleetbase to call your endpoint. You maintain the data updates, storage, compute, monitoring and backups.

For a Uganda-focused deployment where you want to control routing data, self-hosting is the target described below.

## 5. Self-host Valhalla for Uganda

### 5.1 Check host capacity and port availability

Valhalla must build a routing graph from the OpenStreetMap PBF extract. The graph and tile storage require considerably more disk and memory than the compressed PBF file itself. The Uganda extract is hundreds of megabytes; reserve additional disk space and memory for graph generation and runtime. Tile generation can take time.

Check that host port 8002 is free:

```bash
sudo ss -ltnp '( sport = :8002 )'
```

If another service uses it, choose a different host port and use that port consistently in the endpoint configuration.

### 5.2 Create a persistent data directory and download Uganda data

From the Fleetbase project directory:

```bash
cd ~/bwongo-digital-solutions/fleetbase
mkdir -p valhalla/custom_files
cd valhalla/custom_files
wget -O uganda-latest.osm.pbf \
  https://download.geofabrik.de/africa/uganda-latest.osm.pbf
cd ../..
```

Geofabrik provides the Uganda OpenStreetMap PBF extract. The extract is updated periodically; check the download page for its current timestamp and size. OpenStreetMap-derived data is subject to the Open Database License; provide appropriate attribution and review the licence obligations for your use.

### 5.3 Start the Valhalla Docker service

The official Valhalla scripted image builds routing tiles from PBF files found in its persistent `/custom_files` directory and serves the HTTP API on port 8002.

For an initial isolated test, start it with:

```bash
docker run -d \
  --name fleetbase-valhalla \
  --restart unless-stopped \
  -p 8002:8002 \
  -v "$PWD/valhalla/custom_files:/custom_files" \
  ghcr.io/valhalla/valhalla-scripted:latest
```

Run this command from the Fleetbase project directory so the volume path resolves correctly. Keep the directory persistent; it contains the PBF input and generated routing data.

Watch the first graph build:

```bash
docker logs -f fleetbase-valhalla
```

Wait until tile generation has finished and the service is listening before testing it. Do not assume the server is ready immediately after the container starts.

For a permanent deployment, it is usually cleaner to define Valhalla as a service in a Compose override so it shares Fleetbase's Docker network. In that arrangement the API container can address it by the Compose service name, e.g. `http://valhalla:8002`. Do not use `localhost:8002` from inside the API container: localhost there means the API container itself.

### 5.4 Verify the Valhalla service

Check the container and logs:

```bash
docker ps --filter name=fleetbase-valhalla
docker logs --tail=100 fleetbase-valhalla
```

A Valhalla route request uses the `/route` endpoint. Example test coordinates below are in Kampala; replace them with known, safe test points in your service area:

```bash
curl -sS http://localhost:8002/route \
  -H 'Content-Type: application/json' \
  -d '{
    "locations": [
      {"lat": 0.3136, "lon": 32.5811},
      {"lat": 0.3476, "lon": 32.5825}
    ],
    "costing": "auto",
    "units": "kilometers"
  }'
```

A successful response should contain route/trip information. If the request fails, inspect the Valhalla logs and confirm the graph build completed.

### 5.5 Connect Fleetbase to Valhalla

First confirm that the installed Fleetbase version has the Valhalla routing extension installed and enabled. The extension provides the Valhalla provider integration and the organisation-level override UI. If Valhalla is absent from the routing selector, do not assume that adding an endpoint alone will install the extension.

In the Fleetbase Console:

1. Open **Admin → Fleet-Ops Config → Routing**.
2. Set the system-wide Valhalla endpoint to the URL reachable from the Fleetbase API service. For a Compose service named `valhalla`, use `http://valhalla:8002`.
3. Save the system configuration.
4. Open **Fleet-Ops → Settings → Routing**.
5. Set **Routing Service** to **Valhalla**.
6. Set **Route Optimization Engine** to Valhalla if you also want Valhalla to reorder stops. Otherwise leave the current optimizer unchanged.
7. Set the distance unit to kilometres.
8. Save changes.

The exact menu labels and availability depend on the Fleetbase/Fleet-Ops and Valhalla extension versions installed. The current Fleetbase routing documentation says the system endpoint is configured under Admin → Fleet-Ops Config → Routing; organisation overrides appear on the organisation Routing page only when the Valhalla extension is installed.

### 5.6 Private Docker endpoints and security

A Docker service URL such as `http://valhalla:8002` is a private network endpoint. Some newer Fleetbase releases reject private-address **organisation-level** routing overrides by default. Their release notes describe the `VALHALLA_ALLOW_PRIVATE_HOSTS=true` setting for allowing such organisation overrides. System-level hosts are treated separately.

Before adding that variable, check the release notes and configuration supported by the exact installed Valhalla extension/API version. Do not expose Valhalla's port publicly just to work around an internal Docker networking issue. Prefer a private Compose network and restrict host port publication to localhost or remove it entirely once direct host testing is complete.

### 5.7 Validate real Uganda routes

Test multiple known routes before operational use:

- Short urban routes in Kampala.
- Kampala to Entebbe.
- Routes crossing urban and rural areas.
- Routes with multiple stops.
- Routes in each district where the service will operate.

Compare road choice, distance and ETA with local knowledge. OpenStreetMap completeness varies by area, and Valhalla does not provide live traffic unless an appropriate traffic data source is configured. Do not use unvalidated ETAs for customer promises or automated dispatch decisions.

### 5.8 Update the Uganda map data

The `latest` PBF file changes over time. Plan a controlled update process:

1. Download a new PBF file to a staging directory.
2. Keep the existing working graph and PBF as a rollback copy.
3. Check the new extract timestamp and checksum.
4. Replace the staged PBF and rebuild tiles during a maintenance window.
5. Test representative routes.
6. Promote the new graph only after validation.

Do not delete the current graph before a replacement has built and passed tests.

## 6. Routing alternatives

| Option | Routing / map drawing | Stop optimisation | Notes |
|---|---|---|---|
| Valhalla | Yes | Yes | Open-source; supports multiple travel modes; can be self-hosted |
| OSRM | Yes | Yes, via trip endpoint | Open-source; self-hosted; configure host at platform level |
| Google Maps | Yes | Yes | External provider; requires Google Maps configuration and is subject to billing/quotas |
| VROOM | No | Yes | Dedicated vehicle-routing optimiser; pair it with a routing service such as Valhalla or OSRM |

## 7. Production and LAN considerations

- For access from another computer, do not configure browser-facing API URLs as `localhost`; use the Docker host's LAN IP or a DNS name with HTTPS.
- Keep internal Docker service URLs (such as `http://valhalla:8002`) separate from browser-facing URLs.
- Use HTTPS, restrict exposed ports, protect admin accounts, configure backups, and monitor database, queue and scheduler health before production.
- Review the OpenStreetMap attribution and ODbL requirements when using Geofabrik extracts.
- Confirm Fleetbase component and extension compatibility before upgrading. The deployment discussed here showed Console v0.7.75 and API v0.8.2; do not assume newer routing instructions map exactly to that combination.

## 8. References

- [Fleetbase Routing Settings](https://fleetbase.io/docs/fleet-ops/settings/routing)
- [Fleetbase Valhalla extension](https://github.com/fleetbase/valhalla)
- [Fleetbase release notes](https://github.com/fleetbase/fleetbase/releases)
- [Valhalla Docker image guide](https://github.com/valhalla/valhalla/blob/master/docker/README.md)
- [Valhalla installation documentation](https://valhalla.github.io/valhalla/start/installation/)
- [Geofabrik Uganda OpenStreetMap extract](https://download.geofabrik.de/africa/uganda.html)
- [Geofabrik data and licence information](https://www.geofabrik.de/data/download.html)
