# Challenge 4: Node and Storage Issues

## Overview

During Challenge 4, the Kubernetes worker node `sanjay-challenge-worker2` became unavailable and entered a `NotReady` state. This affected scheduling and overall cluster health.

The issue required node-level troubleshooting inside the KIND worker container.

---

# Symptoms Observed

## Node Status Issues

The worker node showed the following status:

```bash
kubectl get nodes
```

Output:

```bash
sanjay-challenge-worker2   NotReady,SchedulingDisabled
```

---

## Additional Problems Observed

* Pods were not scheduling on `worker2`
* Node had taints:

  * `node.kubernetes.io/unreachable`
  * `node.kubernetes.io/unschedulable`
* Kubelet stopped reporting node health
* DNS and networking previously became unstable during troubleshooting

---

# Investigation Steps

## 1. Verified Node Status

```bash
kubectl get nodes
```

---

## 2. Inspected Node Details

```bash
kubectl describe node sanjay-challenge-worker2
```

Important error identified:

```bash
NodeStatusUnknown
Kubelet stopped posting node status.
```

---

## 3. Entered the KIND Worker Container

```bash
docker exec -it sanjay-challenge-worker2 bash
```

---

## 4. Checked Disk Space

```bash
df -h
```

Result:

* Disk space was healthy
* No storage exhaustion issue found

---

## 5. Checked Kubelet Certificates

```bash
ls -la /var/lib/kubelet/pki/
```

Output showed:

```bash
kubelet-client-current.pem.bak
```

But the active kubelet client certificate was missing:

```bash
kubelet-client-current.pem
```

This confirmed that kubelet authentication was failing.

---

# Root Cause Identified

## Root Cause

The kubelet client certificate symlink was missing.

Because of this:

* kubelet could not authenticate with the Kubernetes API server
* node heartbeat stopped
* node entered `NotReady` state
* Kubernetes automatically added unreachable taints
* workloads could not be scheduled

---

# Fix Applied

## Step 1: Recreated the Missing Kubelet Certificate Symlink

Inside the worker container:

```bash
ln -s /var/lib/kubelet/pki/kubelet-client-2026-05-06-03-46-11.pem \
/var/lib/kubelet/pki/kubelet-client-current.pem
```

This recreated the missing kubelet client certificate reference.

---

## Step 2: Restarted Kubelet

```bash
systemctl restart kubelet
```

---

## Step 3: Verified Kubelet Status

```bash
systemctl status kubelet
```

Verification:

```bash
active (running)
```

---

## Step 4: Exited the Container

```bash
exit
```

---

## Step 5: Verified Node Recovery

```bash
kubectl get nodes
```

Successful output:

```bash
sanjay-challenge-worker2   Ready
```

---

# Verification Performed

## Verified All Nodes Healthy

```bash
kubectl get nodes
```

Result:

* Control plane healthy
* All worker nodes healthy
* No nodes in `NotReady` state

---

## Verified Cluster Networking

```bash
kubectl exec debug-client -- nslookup task-3.t3.svc.cluster.local
```

DNS resolution worked successfully.

---

## Verified Service Connectivity

```bash
kubectl exec debug-client -- curl http://task-3.t3.svc.cluster.local
```

Result:

* Successfully received nginx response
* Cluster networking fully operational

---

# Challenges Faced During Troubleshooting

## 1. Repeated Node Failures

The node stayed in `NotReady` state even after:

* restarting Docker container
* restarting kube-proxy
* restarting CoreDNS
* deleting system pods

This made the issue difficult to isolate initially.

---

## 2. DNS Failures During Investigation

DNS resolution failed temporarily while troubleshooting.

This caused:

```bash
Could not resolve host
```

which initially appeared like a networking issue.

Later investigation confirmed CoreDNS misconfiguration and network policies were separate issues from the worker node problem.

---

## 3. Confusion Between Kubernetes and Node-Level Issues

At first, the issue looked like:

* Kubernetes scheduling issue
* networking issue
* CoreDNS issue

But detailed node inspection confirmed it was actually a kubelet authentication issue caused by the missing certificate symlink.

---

# Lessons Learned

## Key Learnings

* Always inspect node details using:

```bash
kubectl describe node <node-name>
```

* `NodeStatusUnknown` usually indicates kubelet communication problems.

* KIND nodes are Docker containers, so low-level debugging can be done using:

```bash
docker exec -it <node-name> bash
```

* Missing kubelet certificates can completely stop node communication with the control plane.

* Restarting Kubernetes components blindly is not always effective.

* Root cause analysis is important before performing cluster-wide changes.

---

# Final Result

Successfully restored:

* kubelet authentication
* node heartbeat communication
* node scheduling
* overall cluster stability

Final cluster status:

```bash
All nodes in Ready state
```

Challenge 4 completed successfully.

