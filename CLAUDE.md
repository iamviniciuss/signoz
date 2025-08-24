# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

This repository contains a SigNoz observability stack setup using Docker Compose. SigNoz is an open-source APM and observability platform that provides metrics, traces, and logs in a single application.

## Architecture

The setup consists of the following core services:

### Core Services
- **SigNoz (signoz-2)**: Main application server on port 8080, handles UI and API
- **ClickHouse (clickhouse)**: Time-series database for storing observability data
- **ZooKeeper (zookeeper-1-beta)**: Coordination service for ClickHouse clustering
- **OpenTelemetry Collector (otel-collector)**: Receives telemetry data via OTLP (ports 4317/4318)

### Data Flow
1. Applications send telemetry data to OpenTelemetry Collector
2. Collector processes and exports data to ClickHouse databases:
   - `signoz_traces` - distributed tracing data
   - `signoz_metrics` - metrics and span metrics
   - `signoz_logs` - log data
3. SigNoz queries ClickHouse for visualization and alerting

## Common Commands

### Start/Stop Services
```bash
# Start all services
docker-compose up -d

# Stop all services
docker-compose down

# View service logs
docker-compose logs -f [service-name]

# Check service status
docker-compose ps
```

### Service Management
```bash
# Restart specific service
docker-compose restart [service-name]

# Rebuild and restart after config changes
docker-compose up -d --force-recreate [service-name]

# Scale services (if configured)
docker-compose up -d --scale [service-name]=N
```

### Database Operations
```bash
# Connect to ClickHouse client
docker-compose exec clickhouse clickhouse-client

# Check ClickHouse health
docker-compose exec clickhouse wget --spider -q localhost:8123/ping
```

## Configuration Structure

### Docker Compose Configuration
- **Volumes**: Persistent storage for ClickHouse data, SQLite, and ZooKeeper
- **Networks**: All services communicate via `signoz_setup_network`
- **Health checks**: Services have dependency chains with health check requirements
- **Volume mounts**: ClickHouse configs are mounted from `./common/clickhouse/`

### ClickHouse Configuration
- **Main config**: `common/clickhouse/config.xml` - server settings, ports, logging
- **Cluster config**: `common/clickhouse/cluster.xml` - defines cluster topology
- **Users config**: `common/clickhouse/users.xml` - authentication and permissions
- **Custom functions**: `common/clickhouse/custom-function.xml` - UDFs for SigNoz

### OpenTelemetry Collector
- **Primary config**: `otel-collector-config.yaml` - receivers, processors, exporters
- **OpAMP config**: `otel-collector-opamp-config.yaml` - remote management endpoint
- **Receivers**: OTLP (gRPC/HTTP), Prometheus scraping
- **Exporters**: Multiple ClickHouse exporters for different data types

### Prometheus Configuration
- **Config file**: `prometheus.yml` - minimal config for remote read from ClickHouse
- **Remote read**: Configured to read metrics from ClickHouse via SigNoz

## Development Notes

### Port Configuration
- SigNoz UI: 8080
- ClickHouse HTTP: 8123 (internal)
- ClickHouse Native: 9000 (internal)
- OTLP gRPC: 4317
- OTLP HTTP: 4318
- ZooKeeper: 2181 (internal)

### Environment Variables
- `VERSION`: SigNoz version (default: v0.87.0)
- `OTELCOL_TAG`: OpenTelemetry Collector version (default: v0.111.42)
- Various ClickHouse DSN and database paths in signoz-2 service

### Volume Considerations
- ClickHouse data persists in `signoz-clickhouse-2` volume
- SigNoz metadata in `signoz-sqlite` volume
- ZooKeeper data in `signoz-zookeeper-1-beta` volume
- Local paths are hardcoded in compose file and may need adjustment

### Schema Migrations
- `schema-migrator-sync-2`: Runs synchronous migrations on startup
- `schema-migrator-async-2`: Handles background schema updates
- Both depend on ClickHouse health checks

## Troubleshooting

### Service Dependencies
Services start in order: init-clickhouse → zookeeper → clickhouse → schema-migrators → signoz → otel-collector

### Common Issues
- Check ClickHouse is healthy before starting dependent services
- Verify volume mount paths exist and are accessible
- Ensure network connectivity between services
- Monitor resource usage as ClickHouse can be memory-intensive

### Health Check Endpoints
- SigNoz: `localhost:8080/api/v1/health`
- ClickHouse: `localhost:8123/ping`
- ZooKeeper: `localhost:8080/commands/ruok`