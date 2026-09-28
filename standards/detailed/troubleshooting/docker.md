# Docker Troubleshooting

Start with daemon health, container state, recent logs, and published ports:

```bash
# Check the client and daemon
docker version
docker info

# Find stopped or unhealthy containers
docker ps -a
docker inspect CONTAINER

# Read recent logs
docker logs --tail 100 CONTAINER

# Check host-to-container port mappings
docker port CONTAINER
```

If a container exits, inspect its logs and exit code before restarting it. If a published port is missing or unreachable, verify the container's listening address and the host-to-container mapping.

For Compose and local Kubernetes setup examples, see [Developer Experience](../../overall/devops.md) and [Getting Started](../../overall/sample-app.md).
