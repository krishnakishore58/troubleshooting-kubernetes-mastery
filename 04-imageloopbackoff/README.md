# ImageLoopBackOff — Troubleshooting the Top 3 Causes

A hands-on demo project for diagnosing and fixing `ImageLoopBackOff` on a local
**kind** cluster running under **WSL2**: an intro, a repeatable diagnostic methodology, three real-world
scenarios (each with a broken manifest, a diagnosis, and a fix), and a wrap-up.

---

## What is ImageLoopBackOff?

`ImageLoopBackOff` (or `ErrImagePull`) is **not** a root cause — it's Kubernetes telling you the
kubelet **failed to pull the container image** and is now **backing off** (waiting longer and longer)
between retry attempts. The back-off is exponential: **5s → 10s → 20s → … → capped at ~5 minutes**.

Your job is to find *why* the image pull failed. The pull error message, the pod events, and the kubelet
logs almost always tell you.

---

## Read the pull error like a pro:

| Error message | Meaning | Typical cause |
|---------------|---------|---------------|
| `manifest not found` | Image tag does not exist in registry | Typo in tag, tag was deleted, wrong registry |
| `no basic auth credentials` | Private registry without credentials | Missing `imagePullSecrets`, secret in wrong namespace |
| `context deadline exceeded` | Pull timed out (> ~2 min) | Large image, slow registry, network latency, `imagePullPolicy: Always` |
| `dial tcp: lookup <host>: no such host` | Registry hostname cannot be resolved | DNS issue, internal registry not reachable from node |
| `connection refused` | Registry host reachable but port closed | Firewall, registry not running, wrong port |
| `TLS handshake timeout` | TLS negotiation timed out | Slow registry, network issues, certificate problems |

---

## The three scenarios

| # | Folder | Root cause | Signature |
|---|--------|-----------|-----------|
| 1 | [`01-image-not-found`](01-image-not-found/) | Image tag typo or does not exist in registry | `manifest not found` |
| 2 | [`02-image-pull-credentials`](02-image-pull-credentials/) | Private registry without imagePullSecrets or wrong credentials | `no basic auth credentials` or `401 Unauthorized` |
| 3 | [`03-registry-unreachable`](03-registry-unreachable/) | Registry unreachable (network, DNS, firewall) | `dial tcp: lookup` / `connection refused` / `context deadline exceeded` |

---

## Quick demo loop (per scenario)

```bash
cd 01-image-not-found      # (or 02-image-pull-credentials, 03-registry-unreachable)

# Apply broken manifest — pod will get stuck in ImagePullBackOff
kubectl apply -f broken.yaml
kubectl -n imageloop-demo get pods -w          # watch pod status (Ctrl-C to stop)

# Diagnose the issue
kubectl -n imageloop-demo describe pod <pod-name>
kubectl -n imageloop-demo get pod <pod-name> -o jsonpath='{.status.containerStatuses[0].state.waiting.message}'
kubectl -n imageloop-demo get events --sort-by='.lastTimestamp' | tail -20

# Apply the fix
kubectl apply -f fixed.yaml
kubectl -n imageloop-demo get pods -w          # watch pod transition to Running

# Cleanup
kubectl delete -f fixed.yaml -f broken.yaml --ignore-not-found
```
---

## Key takeaways

1. **Always read the error message** — Kubernetes tells you *why* the pull failed.
2. **Check DNS first** — If the registry hostname can't be resolved, nothing else matters.
3. **Verify credentials** — Private registries need `imagePullSecrets`; double-check the namespace.
4. **Use fast, reliable registries** — Docker Hub, ECR, GCR are all good defaults. Avoid slow internal registries unless necessary.
5. **Pre-pull images on nodes** — For large or slow-to-pull images, pre-cache them to avoid pull timeouts.
6. **Monitor & alert** — Set up alerts on `ImagePullBackOff` events in production.

---

## See also

- [Kubernetes image pull policy](https://kubernetes.io/docs/concepts/containers/images/)
- [Kubernetes imagePullSecrets](https://kubernetes.io/docs/tasks/configure-pod-container/pull-image-private-registry/)
- [Kind documentation](https://kind.sigs.k8s.io/)
- [Kubernetes troubleshooting guide](https://kubernetes.io/docs/tasks/debug-application-cluster/)
