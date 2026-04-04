# Grafana Observability

This repository packages a small Grafana observability stack for the Florida Mesh deployment, with log collection, dashboards, and alerting focused on the EMQX broker and Floodgate services behind `mqtt.areyoumeshingwith.us`.

The stack provisions:

- Grafana for dashboards and alerting
- Loki for log storage and querying
- Grafana Alloy for Docker log discovery, relabeling, and forwarding

Logs are collected from the Docker socket, labeled with Compose metadata, shipped to Loki, and surfaced in prebuilt Grafana dashboards and alert rules for the Florida Mesh MQTT environment.

## What This Repository Contains

- `docker-compose.yml`: local stack definition for Loki, Alloy, and Grafana
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

The included dashboards and alert rules are tuned around that environment, especially:

- MQTT broker authentication and authorization failures
- unexpected denials on Florida topic paths under `msh/US/FL/#`
- Floodgate service availability, stats emissions, and error lines

## Provisioned Content

### Dashboards

- `EMQX Observability` (`emqx-observability`)
- `Floodgate Observability` (`floodgate-observability`)

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

## Prerequisites

- Docker
- Docker Compose

The stack expects Grafana admin credentials in `.env`. For CI and local testing, `.env.test` is provided.

Example values:

```env
GF_SECURITY_ADMIN_USER=admin
GF_SECURITY_ADMIN_PASSWORD=change-this-now
```

## Getting Started

Use the test environment values:

```bash
ln -sf .env.test .env
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

The Compose stack includes healthchecks for all three services:

- Loki validates its config with the Loki binary
- Alloy checks its readiness endpoint on port `12345`
- Grafana checks `/api/health` on port `3000`

Compose also waits for Loki to become healthy before starting Alloy and Grafana.

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

The Alloy config deliberately keeps labels low-cardinality and leaves most structured fields in the log body for query-time parsing in Loki.

## CI Validation

The GitHub Actions workflow in `.github/workflows/compose-validate.yml` validates that:

- the Compose file renders successfully
- the stack starts successfully
- all services report healthy
- Grafana provisions the dashboards defined in `grafana/dashboards/`
- provisioned dashboards can be retrieved through the Grafana HTTP API

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
- This repository is focused on observability infrastructure and provisioned assets for Florida Mesh; it does not include the EMQX or Floodgate application services themselves.
