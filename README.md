# Docker Toolkit

Simple, reusable Docker Compose setups for common DevOps infrastructure services.
Each service has its own folder with a `compose.yaml`, a `.env.example` and a short README,
so you can copy a folder, edit a few settings and start it.

## Principles

- **Simple and practical.** No unnecessary components, files or scripts.
- **Official images and Docker Compose only.** A Dockerfile appears only when truly needed. No Kubernetes or Swarm.
- **Pinned versions.** No `latest` tags. Update versions deliberately.
- **Essential security only.** No hardcoded passwords, no unnecessary published ports, persistent storage where needed.
- **Configurable.** Settings live in `.env`. Optional features are commented blocks in `compose.yaml`.
- **Honest about limits.** Each README says what the setup is not (for example, replication is not high availability).


More services are planned (PostgreSQL, MariaDB, Redis, MongoDB, ClickHouse, RabbitMQ, Kafka, Nginx, HAProxy,
Traefik, Prometheus, Grafana, Loki, Alertmanager, Jenkins, Nexus, MinIO, Keycloak, n8n and others).

## Quick start

```bash
git clone https://github.com/abe-solhjoo/devops-docker-toolkit.git
cp .env.example .env
# edit .env (at least the password)
docker compose up -d
docker compose ps
```

Then follow the README inside the service folder.

## Before using in production

- Read the "Production considerations" section of the service README.
- Check the current patch release of the pinned image and update it deliberately.
- Test the setup, and test your backup restore, in a non-production environment first.
- These files are provided as examples and starting points. You are responsible for reviewing them against your own requirements.

## Contributing

Issues and pull requests are welcome. Please keep changes simple, follow the principles above,
and do not claim a configuration is tested unless you ran it.

## License

[MIT](LICENSE)
