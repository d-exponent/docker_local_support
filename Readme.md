# Local Development Infrastructure

Docker Compose setup for the supporting services used during local development: a database, a cache, and a monitoring stack. The application servers themselves are **not** part of this stack.

## Purpose

- Provide Postgres (with PostGIS) and Redis for the apps without installing them locally.
- Provide pgAdmin for inspecting the database.
- Provide Prometheus and Grafana to monitor the apps servers (REST API and realtime).. More will be added ...

## Services

| Service    | Image                      | Host port | Purpose                         |
|------------|----------------------------|-----------|---------------------------------|
| Postgres   | `postgis/postgis:17-3.5`   | 5432      | Primary database                |
| pgAdmin    | `dpage/pgadmin4`           | 5003      | Database admin UI               |
| Redis      | `redis:7-alpine`           | 6379      | Cache / pub-sub                 |
| Prometheus | `prom/prometheus`          | 9090      | Metrics collection              |
| Grafana    | `grafana/grafana`          | 3000      | Metrics dashboards              |

## Project structure

```
.
├── docker-compose.yml
├── prometheus/
│   └── prometheus.yml                         # Scrape targets for Prometheus
└── grafana/
    └── provisioning/
        └── datasources/
            └── datasource.yml                 # Auto-registers Prometheus in Grafana
```

## Getting started

Prerequisites: Docker and Docker Compose.

```bash
docker compose up -d      # start everything
docker compose ps         # check status
docker compose down       # stop (data volumes are kept)
docker compose down -v    # stop and delete all data volumes
```

## Monitoring

Prometheus scrapes the two application servers running on the host:

| Job        | Target                          | App variable      |
|------------|---------------------------------|-------------------|
| `rest-api` | `host.docker.internal:9000`     | `REST_API_PORT`   |
| `realtime` | `host.docker.internal:9001`     | `REALTIME_PORT`   |

Because the servers run outside the Docker network, Prometheus reaches them through `host.docker.internal`. The `extra_hosts` entry in `docker-compose.yml` makes this work on Linux. Docker Desktop on Mac and Windows provides it automatically.

If you change the app ports, update the targets in `prometheus/prometheus.yml` to match.

### Access

- Prometheus UI: http://localhost:9090 (target health at `/targets`)
- Grafana: http://localhost:3000 (default login in the compose file, see Security below)

The Prometheus datasource is provisioned automatically in Grafana, so no manual setup is needed.

### Current status

The apps do not expose a `/metrics` endpoint yet, so both application targets will show as **DOWN** in Prometheus. This is expected. Once metrics are exposed, they will turn **UP** without any changes to this stack, provided the path is `/metrics` (otherwise adjust `metrics_path`).

### Requirements for the apps

- Expose a Prometheus-format endpoint at `/metrics` on each server.
- Listen on `0.0.0.0` rather than only `127.0.0.1`, otherwise the Prometheus container cannot connect.

## Security

This setup is intended for **local development only**. Do not expose it as-is.

- Postgres uses `POSTGRES_HOST_AUTH_METHOD: trust` (no password).
- pgAdmin and Grafana use placeholder credentials set in `docker-compose.yml`.

Change these before using any part of this configuration in a shared or production environment.

## Troubleshooting

- **Targets show DOWN:** confirm the app is running, exposes `/metrics`, and listens on `0.0.0.0`.
- **`host.docker.internal` not resolving (Linux):** make sure the `extra_hosts` entry is present on the `prometheus` service.
- **Grafana shows no datasource:** check that `grafana/provisioning/datasources/` is mounted and contains a valid YAML file.