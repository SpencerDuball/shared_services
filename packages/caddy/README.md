# Caddy

This package holds the `Caddyfile` to be setup as a config in Docker Swarm.

## Rotating

The way Docker handles rotating configs/secrets is a bit of a pain. All configs/secrets are immutable, and the only way for these to be rotated is by:

- creating a new config/secret (with a different name)
- updating the service by removing the old config/secret and adding the new config/secret simultaneously
- manually deleting the old config/secret

```bash
# create the new config (caddyfile.v1) from the file (Caddyfile)
docker config create caddyfile.v1 Caddyfile

# update a service with the CLI (It will restart the container!)
docker service update \
--config-rm caddyfile.v0 \
--config-add source=caddyfile.v1,target=/etc/caddy/Caddyfile
shared_services_caddy

# remove the old config
docker config rm caddyfile.v0
```

This works, but now our `compose.prod.yaml` file is out-of-sync with the actual secret that is deployed on our swarm. However, there is a way for us to make this a little bit more user friendly - and it means implementing a secret naming strategy so we can update these values easily.

The strategy deployed in this project combines some planning in a naming convention for the secrets along with environment variables that are used to update the stack. This naming convention + environment variables means we get the benefits of using a CLI since we don't need to update and make a commit for the compose file on each rotation, plus we get the benefits of using the compose file to perform the service updates.

```yaml
configs:
    caddyfile:
        external: true
        name: "shared_services.caddyfile.${CADDY_ROTATION}"

secrets:
    minio_root_user:
        external: true
        name: "shared_services.minio_root_user.${MINIO_ROTATION}"
    minio_root_password:
        external: true
        name: "shared_services.minio_root_password.${MINIO_ROTATION}
```

Now we can perform rotations with the docker compose file using the environment variables, or like before with the CLI.

```bash
CADDY_ROTATION=v1 MINIO_ROTATION=v4 docker stack deploy -c compose.prod.yaml shared_services
```

# References

- [Rotating Your Docker Secrets Can Be Easy, If You Plan For It](https://anthonymineo.com/rotating-your-docker-secrets-can-be-easy-if-you-plan-for-it/)
- [Compose Configs Spec](https://github.com/compose-spec/compose-spec/blob/main/08-configs.md)
- [Docker Swarm Docs](https://docs.docker.com/engine/swarm/configs/#example-rotate-a-config)

[Cloudflare & Caddyserver TLS](https://samjmck.com/en/blog/using-caddy-with-cloudflare/)

- Document how each of these configurations were created in the docs, how each of the environment variables were chosen and how to re-set this back up (or how to set this up for someone coming in from scratch).
