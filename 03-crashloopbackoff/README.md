# CrashLoopBackOff — Troubleshooting the Top 5 Causes

A hands-on demo project for diagnosing and fixing `CrashLoopBackOff` on a local
**kind** cluster running under **WSL2**: an intro, a repeatable diagnostic methodology, five real-world
scenarios (each with a broken manifest, a diagnosis, and a fix), and a wrap-up.

---

## What is CrashLoopBackOff?

`CrashLoopBackOff` is **not** a root cause — it's Kubernetes telling you a
container keeps **starting, failing, and being restarted**, and the kubelet is
now **backing off** (waiting longer and longer) between restart attempts. The
back-off is exponential: **10s → 20s → 40s → … → capped at 5 minutes**.

Your job is to find *why* the container exits. The exit code, the previous
logs, and the pod events almost always tell you.

---
Create the shared namespace once:

```bash
kubectl apply -f namespace.yaml
kubectl config set-context --current --namespace=crashloop-demo   # optional convenience
```

---
**Read the exit code like a pro:**

| Exit code | Meaning | Typical cause |
|-----------|---------|---------------|
| `0`       | Clean exit | Short-lived process / wrong restartPolicy |
| `1`       | General app error | Unhandled startup exception, failed check |
| `126`     | Not executable | Entrypoint found but no exec permission |
| `127`     | Command not found | Wrong `command`/entrypoint, bad `$PATH` |
| `137`     | SIGKILL (128+9) | **OOMKilled** or forced kill |
| `143`     | SIGTERM (128+15) | Graceful shutdown / evicted |

---

## The five scenarios

| # | Folder | Root cause | Signature |
|---|--------|-----------|-----------|
| 1 | [`01-application-error`](01-application-error/) | App exits non-zero at startup | `Exit Code: 1` + app log |
| 2 | [`02-misconfiguration`](02-misconfiguration/) | Wrong command + missing env var | `Exit Code: 127`, `not found` |
| 3 | [`03-liveness-probe`](03-liveness-probe/) | Misconfigured liveness probe kills a healthy app | `Liveness probe failed` in Events, clean app logs |
| 4 | [`04-oomkilled`](04-oomkilled/) | Memory limit too low | `Reason: OOMKilled`, `Exit Code: 137` |
| 5 | [`05-missing-dependency`](05-missing-dependency/) | Required config file never mounted | log names a missing file, no `volumeMounts` |
---
## Quick demo loop (per scenario)

```bash
cd 01-application-error      # (or 02-, 03-, 04-, 05-)

kubectl apply -f broken.yaml
kubectl -n crashloop-demo get pods -w          # watch it fall into CrashLoopBackOff (Ctrl-C to stop)

kubectl -n crashloop-demo describe pod <pod-name>
kubectl -n crashloop-demo logs pod <pod-name> --previous

kubectl apply -f fixed.yaml
kubectl -n crashloop-demo get pods -w          # watch it go Running

kubectl delete -f fixed.yaml -f broken.yaml --ignore-not-found
```
---

## Full cleanup

```bash
#just nuke the namespace (removes all demo resources at once)
kubectl delete -f namespace.yaml --ignore-not-found
```
