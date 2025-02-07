# Overview

This repository holds the Docker Swarm stack for shared services in my home network. There are a few services which all applications need, support multiple tenants, and are best run as distributed applications for high availability. In cases like this, it is better practice to share these resources rather than have a distributed cluster for each app that might need these services:

- [Caddyserver](https://github.com/caddyserver/caddy) - A reverse proxy which serves as the public endpoint for all web traffic.
- [Minio](https://github.com/minio/minio) - This is an S3 compatible object storage solution for files/images/logs/backups/etc.
- [Grafana LGTM+ Stack](https://grafana.com/about/grafana-stack/) - A complete observability framework for performance testing, application observability, infrastructure observability, and incident response management.
  - [Grafana Alloy](https://grafana.com/docs/alloy/latest/introduction/) - An agent that runs on each node in your deployment. Alloy collects metrics & logs from your applications, transforms them, and then sends them to thier destination. For metrics this will be Grafana Mimir and for logs this will be Grafana Loki.
  - [Grafana Mimir](https://grafana.com/docs/mimir/latest/) - A time-series database used to collect and query metrics data. This is typically things like CPU usage, RAM usage, error/success rates, response times etc.
  - [Grafana Loki](https://grafana.com/docs/loki/latest/get-started/) - A database used to collect, index, query and store logs. This is a single location where you can view all logs in aggregate and search them on specific indexes.

# Deploying

This stack uses [docker secrets](https://docs.docker.com/engine/swarm/secrets/) and [docker configs](https://docs.docker.com/engine/swarm/configs/) to handle sensitive data, or configuration data that may change and not require rebuilding images. Managing secrets and configs requires a bit of preplanning to handle when rotations or updates are necessary, and before we can deploy the stack these values must be created in Docker Swarm.

## Secrets and Configs

To deploy this stack we can first look at the `docker-compose.yaml`
