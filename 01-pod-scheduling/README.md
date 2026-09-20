# Demo execution

This README contains the commands and steps to demo taints & tolerations, nodeSelector, and nodeAffinity on a local kind cluster (WSL2).

## Taint and toleration

Apply taint to the first node (NoExecute example), watch pod behavior and events:

```bash
NODE=$(kubectl get nodes -o jsonpath='{.items[0].metadata.name}')
kubectl taint nodes $NODE maintenance=true:NoExecute

kubectl apply -f pod-no-toleration.yaml

vi pod-no-toleration.yaml

#Apply toleration to the pod
cat >> pod-no-toleration.yaml

  tolerations:
  - key: "maintenance"
    operator: "Equal"
    value: "true"
    effect: "NoExecute"

kubectl apply -f pod-no-toleration.yaml

#Verify the pod is running and is sitting on the tainted node
kubectl -n taint-toleration-demo get pod -o wide
```

Note: Use this single YAML + taint command to demo NoSchedule / PreferNoSchedule semantics by changing the effect in the taint command.

## Node‑selector

Recreate the kind cluster with multiple nodes (example workflow):

```bash
# Delete existing cluster and recreate with multi-node config
kind delete cluster --name kind-demo
vi /mnt/f/software/multi-node-cluster.yaml   # add second worker node in the config
kind create cluster --name kind-demo --config /mnt/f/software/multi-node-cluster.yaml

# Verify nodes and label them
kubectl get nodes
kubectl label node kind-demo-worker size=Large
kubectl label node kind-demo-worker2 size=Medium

# Inspect node details
kubectl describe node kind-demo-worker
kubectl describe node kind-demo-worker2
```

Deploy the pod using nodeSelector and verify scheduling:

```bash
kubectl get pods -o wide
kubectl describe pod nginx-node-selector | sed -n '/Node:/,/Events:/p'
```

## Node‑affinity

Check pod placement and simulate label removal to force rescheduling:

```bash
kubectl -n taint-toleration-demo get pod -o wide

# If the pod is on worker-node2 and you want to force it to move:
kubectl label node kind-demo-worker2 size-
kubectl delete -f pod-with-node-affinity.yaml
kubectl apply -f pod-with-node-affinity.yaml   # pod should schedule on worker-node-1 (if labels match)
```

Notes
- `nodeSelector` is strict: the pod will remain `Pending` if no node has the required label.
- Use `nodeAffinity` for richer rules (preferred/required) and topology awareness.
- Combine positive selection (nodeSelector/nodeAffinity) with taints+tolerations for strong node isolation: label to attract the intended pods, taint to repel unintended pods, and add tolerations only to allowed pods.
- Monitor scheduling events (`kubectl describe pod ...`) to diagnose Pending pods.

Cleanup hints
- Remove labels: `kubectl label node <node-name> size-`
- Remove taints: `kubectl taint nodes <node> maintenance=true:NoExecute-`
- Delete demo resources as needed: `kubectl delete

