# Grafana Observability

This repository packages a small Grafana observability stack for the Florida Mesh deployment, with log collection, dashboards, and alerting focused on the EMQX broker and Floodgate services behind `mqtt.areyoumeshingwith.us`.

The stack provisions:

- Grafana for dashboards and alerting
- Loki for log storage and querying
- Grafana Alloy for Docker log discovery, relabeling, and forwarding
- Redis for Hubot brain persistence and short-lived caching
- Hubot colocated with Loki for Discord bot access to logs

Logs are collected from the Docker socket, labeled with Compose metadata, shipped to Loki, and surfaced in prebuilt Grafana dashboards and alert rules for the Florida Mesh MQTT environment.

## What This Repository Contains

- `docker-compose.yml`: local stack definition for Loki, Alloy, and Grafana
- Redis and Hubot services colocated with the observability stack
- `alloy/config.alloy`: Docker log discovery, relabeling, parsing, and forwarding rules
- `loki/config.yaml`: single-node Loki configuration backed by a local volume
- `grafana/provisioning/datasources/loki.yaml`: provisioned Loki datasource
- `grafana/provisioning/dashboards/dashboard-provider.yaml`: file-based dashboard provisioning
- `grafana/provisioning/alerting/alert-rules.yaml`: provisioned Grafana alert rules
- `grafana/dashboards/*.json`: prebuilt dashboards
- `.github/workflows/compose-validate.yml`: CI validation for compose startup, health, and dashboard provisioning

## Architecture

The runtime flow is:

1. Alloy discovers Docker containers through `/var/run/docker.sock`.
2. Alloy attaches low-cardinality labels such as `compose_service`, `compose_project`, `container`, and `stream`.
3. Alloy applies targeted parsing for EMQX and Floodgate logs.
4. Alloy pushes logs to Loki at `http://loki:3100/loki/api/v1/push`.
5. Grafana connects to Loki as its default datasource and provisions dashboards and alert rules on startup.

## Deployment Scope

This stack is intended to monitor:

- the Florida Mesh EMQX broker serving `mqtt.areyoumeshingwith.us`
- the associated Floodgate services that support that MQTT deployment
- the MongoDB service used by EMQX for MQTT authentication and authorization

The included dashboards and alert rules are tuned around that environment, especially:

- MQTT broker authentication and authorization failures
- unexpected denials on Florida topic paths under `msh/US/FL/#`
- Floodgate service availability, stats emissions, and error lines
- MongoDB availability, error logs, authentication failures, and EMQX MongoDB driver crashes

Hubot is colocated with Loki so the bot can query the same log store without depending on the EMQX deployment host.

## Provisioned Content

### Dashboards

- `EMQX Observability` (`emqx-observability`)
- `Floodgate Observability` (`floodgate-observability`)
- `MongoDB Observability` (`mongodb-observability`)

### Alert Groups

- `emqx-observability`
  - EMQX unexpected FL topic denials (all actions)
  - EMQX unexpected FL topic denials (non-SUBSCRIBE)
  - EMQX authorization denial spike
  - EMQX error spike
  - EMQX authentication failures detected
  - EMQX no logs received
- `floodgate-observability`
  - Floodgate no logs received
  - Floodgate no stats lines received
  - Floodgate error lines detected
  - Floodgate stats reported errors
- `mongodb-observability`
  - MongoDB no logs received
  - MongoDB error or fatal lines detected
  - MongoDB authentication failures detected
  - EMQX MongoDB driver crashes detected

## Prerequisites

- Docker
- Docker Compose

The stack expects Grafana admin credentials in `.env`. For CI and local testing, `.env.test` is provided.

Example values:

```env
GF_SECURITY_ADMIN_USER=admin
GF_SECURITY_ADMIN_PASSWORD=change-this-now
```

To run Hubot successfully, `.env` also needs:

```env
HUBOT_DISCORD_TOKEN=your-discord-bot-token
HUBOT_NAME=MeshBot
HUBOT_LOG_LEVEL=info
REDIS_URL=redis://redis:6379
LOKI_URL=http://loki:3100
LOKI_USERNAME=
LOKI_PASSWORD=
NODE_LOGS_CACHE_TTL_SECONDS=30
```

## Getting Started

Use the test environment values:

```bash
ln -sf .env.test .env
docker compose up -d loki alloy grafana
```

To start the full deployed stack, including Redis and Hubot:

```bash
docker compose up -d
```

Then open:

- Grafana: `http://localhost:3000`
- Loki: `http://localhost:3100`
- Alloy HTTP endpoint: `http://localhost:12345`

Log into Grafana with the credentials from `.env`.

To stop the stack:

```bash
docker compose down -v
```

## Healthchecks

The observability services include healthchecks for:

- Loki validates its config with the Loki binary
- Alloy checks its readiness endpoint on port `12345`
- Grafana checks `/api/health` on port `3000`

Compose also waits for Loki to become healthy before starting Alloy and Grafana.

- Redis is health-checked with `redis-cli ping`
- Hubot waits for both Loki and Redis before starting

## Log Labeling and Parsing

All logs receive Docker Compose context labels, including:

- `compose_service`
- `compose_project`
- `container`
- `stream`
- `job=docker`

Additional parsing is applied selectively:

- EMQX logs: extracts `level`, `tag`, and `action`
- Floodgate logs: extracts `level`, `logger`, `event`, and structured timestamps
- MongoDB logs: extracts `level` and `component` from MongoDB JSON logs

The Alloy config deliberately keeps labels low-cardinality and leaves most structured fields in the log body for query-time parsing in Loki.

## CI Validation

The GitHub Actions workflow in `.github/workflows/compose-validate.yml` validates that:

- the Compose file renders successfully
- the observability services start successfully
- Loki, Alloy, and Grafana report healthy
- Grafana provisions the dashboards defined in `grafana/dashboards/`
- provisioned dashboards can be retrieved through the Grafana HTTP API

The workflow intentionally does not start Hubot because a real `HUBOT_DISCORD_TOKEN` is not available in CI for this repository.

## Repository Layout

```text
.
├── alloy/
│   └── config.alloy
├── grafana/
│   ├── dashboards/
│   └── provisioning/
├── loki/
│   └── config.yaml
├── .github/
│   └── workflows/
├── .env.test
└── docker-compose.yml
```

## Notes

- Grafana state is stored in the `grafana-data` volume.
- Loki data is stored in the `loki-data` volume.
- Redis brain/cache state is stored in the `redis-data` volume.
- This repository is focused on observability infrastructure and provisioned assets for Florida Mesh; it does not include the EMQX or Floodgate application services themselves.
