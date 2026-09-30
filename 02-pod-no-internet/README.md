# Kind — Demo: Pod has no Internet (NetworkPolicy & CoreDNS)

This README is a concise playbook built from your notes to demo and troubleshoot pod egress issues on a Kind cluster running.

---

## 1) Kind cluster config (multi-node, CNI disabled so you can install your CNI)

Save to: `/tmp/multi-node-cluster.yaml`

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
    extraPortMappings:
      - containerPort: 30000
        hostPort: 30000
        listenAddress: "0.0.0.0"
        protocol: TCP
  - role: worker
networking:
  disableDefaultCNI: true
```

Note: `disableDefaultCNI: true` disables the default CNI so you can install a NetworkPolicy‑capable CNI (e.g. Calico).

---

## 2) Install CNI (Calico example)

Run (requires internet):
```bash
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.30.0/manifests/calico.yaml
kubectl -n kube-system wait --for=condition=Ready pod -l k8s-app=calico-node --timeout=180s || true
kubectl get daemonset -A
kubectl get pods -n kube-system -l k8s-app=calico-node -o wide
```


---

## 3) NetworkPolicy deny-everything demo

Create file `networkpolicy-deny-egress.yaml`:
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-egress-all
  namespace: default
spec:
  podSelector: {}
  policyTypes:
  - Egress
  egress: []
```

Create test pod `sample-pod.yaml`:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: curl-test
  labels:
    app: curl-test
spec:
  containers:
  - name: curl
    image: curlimages/curl:8.3.0
    command: ["sleep","3600"]
  restartPolicy: Never
```

Apply and test:
```bash
kubectl apply -f sample-pod.yaml
kubectl wait --for=condition=Ready pod/curl-test --timeout=60s

# verify internet from pod
kubectl exec -it pod/curl-test -- curl -I https://www.google.com || true

# apply deny policy (should block egress if CNI enforces NetworkPolicy)
kubectl apply -f networkpolicy-deny-egress.yaml
kubectl exec -it pod/curl-test -- curl -I https://www.google.com || true

# remove policy and retest
kubectl delete -f networkpolicy-deny-egress.yaml
kubectl exec -it pod/curl-test -- curl -I https://www.google.com || true
```

If the deny policy has no effect:
- Ensure CNI (Calico/Antrea/Cilium) is installed and healthy.
- Ensure the pod is not using `hostNetwork: true`.
- Ensure the policy selects the pod (labels/namespace).

If cluster is not important, you can `kind delete cluster --name <name>` and recreate.

---

## 4) CoreDNS: backup, break (simulate DNS failure), restore

Backup current CoreDNS config:
```bash
kubectl -n kube-system get configmap coredns -o yaml > /tmp/coredns-orig.yaml
```

Patch CoreDNS to forward to an unreachable IP (simulate DNS failure).
Important: do not include a leading space before `.:53`:
```bash
kubectl -n kube-system patch configmap coredns --type merge -p '{"data":{"Corefile":".:53 {\n    errors\n    forward . 192.0.2.1\n    cache 30\n}\n"}}'
kubectl -n kube-system rollout restart deployment coredns
kubectl -n kube-system rollout status deployment coredns --timeout=60s || true
```

Test DNS/HTTP from pod:
```bash
kubectl exec -it pod/curl-test -- sh -c "nslookup google.com || getent hosts google.com" || true
kubectl exec -it pod/curl-test -- curl -I https://www.google.com || true
```

Restore CoreDNS (using backup):
```bash
kubectl -n kube-system replace --force -f /tmp/coredns-orig.yaml
kubectl -n kube-system rollout restart deployment coredns
kubectl -n kube-system rollout status deployment coredns --timeout=60s || true
```

If backup is missing, apply a safe default Corefile (scripts in repo can automate this).

---

## 5) Useful debugging & checks

- List NetworkPolicies:
  kubectl get networkpolicy --all-namespaces

- Inspect pod and labels:
  kubectl get pod curl-test -o yaml

- DNS service:
  kubectl get svc -n kube-system kube-dns -o yaml || kubectl get svc -n kube-system coredns -o yaml

- Check node iptables (kind node container name may vary):
  docker ps --filter "name=kind" --format '{{.Names}}'
  docker exec -it <kind-node> iptables -t nat -L POSTROUTING -v -n

- For transient demo toggles you can:
  - Delete Calico-created MASQUERADE in cali-POSTROUTING (ephemeral)
  - Patch Calico IPPool natOutgoing=false (persistent)
  - Insert/remove `iptables -I FORWARD -s <POD_CIDR> -j DROP` on the node (reliable demo)

---

## Notes & caveats

- Kubernetes NetworkPolicy uses IPs/CIDRs, not domain names. For domain-based rules use Calico FQDN policies or an egress proxy.
- Changes made directly to node iptables are ephemeral and may be lost when kind node container restarts.
- Always back up `kube-system` ConfigMaps (CoreDNS) before editing.
- If things are irreparably broken, recreate cluster with `kind delete cluster` and `kind create cluster`.
