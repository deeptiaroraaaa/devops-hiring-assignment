# Challenge 3 – Kubernetes Network & DNS Troubleshooting

## Objective

Restore connectivity between the `debug-client` pod and the `task-3` service inside the Kubernetes cluster.

The challenge involved troubleshooting DNS resolution failures, missing debug resources, and restrictive network policies blocking communication between namespaces.

---

# Initial Problems Observed

The following issues were identified during troubleshooting:

1. `debug-client` pod was missing
2. DNS resolution for `task-3.t3.svc.cluster.local` failed
3. Egress traffic was blocked from the `default` namespace
4. Ingress traffic was blocked into the `t3` namespace

---

# Root Cause Analysis

---

## Root Cause 1 — Debug Client Pod Was Not Created

### Problem

The troubleshooting pod required for connectivity testing did not exist inside the `default` namespace.

### Verification

```bash id="a8w1e2"
kubectl get pods -n default
```

Output:

```text id="f6y2k1"
No resources found in default namespace.
```

### Impact

Without the debug pod:

* DNS resolution could not be tested
* Service connectivity checks could not be performed
* Network troubleshooting was blocked

---

## Root Cause 2 — CoreDNS Misconfiguration

### Problem

The CoreDNS ConfigMap contained a malicious rewrite rule redirecting the service DNS query to an invalid domain.

### Verification

DNS lookup failed:

```bash id="u6j8q0"
kubectl exec -it debug-client -n default -- \
nslookup task-3.t3.svc.cluster.local
```

Observed:

* Timeout responses
* NXDOMAIN failures

Curl request also failed:

```bash id="m0r9n4"
kubectl exec -it debug-client -n default -- \
curl http://task-3.t3.svc.cluster.local
```

Output:

```text id="w8t2g9"
curl: (6) Could not resolve host:
task-3.t3.svc.cluster.local
```

Checked CoreDNS configuration:

```bash id="j4v7l2"
kubectl get configmap coredns -n kube-system -o yaml
```

Found invalid rewrite rule:

```text id="c2n6b5"
rewrite name task-3.t3.svc.cluster.local \
task-3.t3.svc.cluster.invalid
```

### Impact

DNS requests for the service were redirected to a non-existent domain, causing all service discovery to fail.

---

## Root Cause 3 — Egress Network Policy Blocking Outbound Traffic

### Problem

A deny-all-egress network policy existed in the `default` namespace.

### Verification

```bash id="h3d5r7"
kubectl describe networkpolicy deny-all-egress -n default
```

Observed:

```text id="e7v4y1"
Allowing egress traffic:
  <none>
```

### Impact

All outbound traffic from pods in the `default` namespace was blocked, including:

* DNS requests
* HTTP requests
* Service communication

---

## Root Cause 4 — Ingress Network Policy Blocking Inbound Traffic

### Problem

A deny-all-ingress policy existed in the `t3` namespace.

### Verification

```bash id="q5z8p2"
kubectl describe networkpolicy deny-all-ingress -n t3
```

### Impact

All incoming traffic to services in the `t3` namespace was blocked, preventing application access even if DNS worked correctly.

---

# Fixes Applied

---

## Fix 1 — Created Debug Client Pod

Created a debugging pod using the `netshoot` image:

```bash id="g7u2m5"
kubectl run debug-client \
  --image=nicolaka/netshoot \
  -n default \
  --labels='role=debug-client' \
  -- sleep infinity
```

### Verification

```bash id="k9x3b8"
kubectl get pods -n default
```

Output:

```text id="n4f1w6"
debug-client   1/1   Running
```

---

## Fix 2 — Restored Correct CoreDNS Configuration

Edited the CoreDNS ConfigMap:

```bash id="p2s7d1"
kubectl edit configmap coredns -n kube-system
```

Removed the malicious rewrite rule:

```text id="r5h9c3"
rewrite name task-3.t3.svc.cluster.local \
task-3.t3.svc.cluster.invalid
```

Restarted CoreDNS pods:

```bash id="z6m1q8"
kubectl delete pods -n kube-system -l k8s-app=kube-dns
```

---

## Fix 3 — Removed Blocking Egress Policy

Deleted restrictive network policy:

```bash id="v8e2t5"
kubectl delete networkpolicy deny-all-egress -n default
```

---

## Fix 4 — Removed Blocking Ingress Policy

Deleted restrictive ingress policy:

```bash id="y1u6r4"
kubectl delete networkpolicy deny-all-ingress -n t3
```

---

# Verification Steps

---

## Verified DNS Resolution

```bash id="d7c4n2"
kubectl exec -it debug-client -n default -- \
nslookup task-3.t3.svc.cluster.local
```

Successful result:

```text id="l5f8b1"
Name: task-3.t3.svc.cluster.local
Address: 10.96.x.x
```

---

## Verified Service Connectivity

```bash id="t4g9w3"
kubectl exec -it debug-client -n default -- \
curl http://task-3.t3.svc.cluster.local
```

Successful output:

```text id="o2m6p7"
HTTP/1.1 200 OK
```

---

## Verified Network Policies Removal

```bash id="i3q7x5"
kubectl get networkpolicy -n default
kubectl get networkpolicy -n t3
```

Output:

```text id="b8k1v4"
No resources found
```

---

# Final Result

All networking and DNS issues were successfully resolved.

### Successfully Fixed

* Debug client pod creation
* CoreDNS DNS rewrite issue
* Egress traffic blockage
* Ingress traffic blockage
* Internal Kubernetes service connectivity

### Final Outcome

✅ DNS resolution restored
✅ Pod-to-service communication restored
✅ HTTP connectivity working successfully
✅ Network policies cleaned up
✅ Cluster networking functioning normally
