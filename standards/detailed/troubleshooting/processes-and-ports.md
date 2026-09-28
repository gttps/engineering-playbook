# Processes, Ports, and Pods

Find which process is listening on a port before changing or stopping it:

```bash
# macOS/Linux: list listening TCP sockets with process IDs
lsof -nP -iTCP:8080 -sTCP:LISTEN

# Linux alternative
ss -ltnp

# macOS/Linux: inspect processes
ps aux
top

# Windows PowerShell: find the process listening on a port
Get-NetTCPConnection -LocalPort 8080 | Select-Object LocalAddress,LocalPort,State,OwningProcess
Get-Process -Id <PID>
```

Check port mappings at the layer where the service runs:

```bash
# Docker host-to-container mappings
docker ps
docker port CONTAINER

# Kubernetes pod IPs and Service ports
kubectl get pods -o wide
kubectl get services,endpoints
kubectl describe service SERVICE
```

Confirm the process listens on the expected interface and port. A service bound only to `127.0.0.1` may not be reachable through a container or pod network. See the [Kubernetes troubleshooting guide](./kubernetes.md) for pod and Service checks.
