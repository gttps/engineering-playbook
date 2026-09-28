# Kubernetes Troubleshooting

Check the current context and namespace first, then inspect workload state, events, logs, and service routing:

```bash
# Confirm cluster and context
kubectl config current-context
kubectl get nodes

# Inspect pods and recent events
kubectl get pods -o wide
kubectl describe pod POD
kubectl get events --sort-by=.metadata.creationTimestamp

# Read current or previous container logs
kubectl logs POD --all-containers
kubectl logs POD --previous

# Check service endpoints and test access locally
kubectl get services,endpoints
kubectl port-forward svc/SERVICE 8080:80
```

For `Pending`, `ImagePullBackOff`, or `CrashLoopBackOff`, use `describe` and events to find the specific scheduling, image, or startup error. Check that a Service selector matches ready pod labels before debugging the network path.

For more detailed guides refer [K8s debugging](https://kubernetes.io/docs/tasks/debug/)

Related: [Local Kubernetes with Kind](../../overall/devops.md) and [Local Kubernetes setup](../../overall/sample-app.md).
