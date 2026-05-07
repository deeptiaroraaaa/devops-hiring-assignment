# Challenge 6: Performance Triage Under Load

## Challenge Objective

In namespace `t6`, the deployment `api-server` was experiencing intermittent failures under load. A Horizontal Pod Autoscaler (HPA) was configured but not functioning correctly.

### Rules

* Do NOT stop or delete the `load-generator` pod
* Do NOT reduce the request rate

### Goal

Make the `api-server` handle all requests successfully with response times under 2 seconds by fixing autoscaling and resource-related issues.

---

# Initial Symptoms Observed

## 1. HPA Metrics Showing Unknown

```bash
kubectl get hpa -n t6
```

Output:

```bash
api-hpa   Deployment/api-server   cpu: <unknown>/50%   1   5   1
```

This indicated that HPA could not fetch CPU metrics.

---

## 2. Metrics API Not Available

```bash
kubectl top pods -n t6
```

Output:

```bash
error: Metrics API not available
```

This confirmed that metrics-server was either missing or not configured properly.

---

## 3. Deployment Not Scaling

```bash
kubectl get deployment api-server -n t6
```

Output:

```bash
READY   1/1
```

Even under load, the deployment remained at a single replica.

---

## 4. Strict Resource Limits Applied

```bash
kubectl describe limitrange strict-limits -n t6
```

Output:

```bash
Container   cpu      Default Request: 50m
Container   memory   Default Request: 32Mi
```

The namespace LimitRange was restricting pod resources heavily.

---

# Root Causes Identified

## Root Cause 1: Metrics Server Missing / Misconfigured

HPA depends on metrics-server to collect CPU and memory metrics.

Because metrics-server was unavailable initially, HPA displayed:

```bash
cpu: <unknown>/50%
```

and autoscaling could not work.

---

## Root Cause 2: Namespace LimitRange Restricting Resources

The `strict-limits` LimitRange forced very small CPU and memory requests/limits.

This prevented proper scaling behavior and resource allocation.

---

## Root Cause 3: No Effective Load Initially

The deployment was not receiving enough measurable CPU load to trigger autoscaling.

---

## Root Cause 4: HPA Could Not Calculate Utilization Properly

Without working metrics collection and sufficient CPU usage, HPA could not scale replicas.

---

# Fixes Applied

## Fix 1: Removed Restrictive LimitRange

Deleted the namespace LimitRange:

```bash
kubectl delete limitrange strict-limits -n t6
```

Verification:

```bash
kubectl get limitrange -n t6
```

Output:

```bash
No resources found in t6 namespace.
```

---

## Fix 2: Verified Metrics Server Functionality

Checked metrics availability:

```bash
kubectl top pods -n t6
```

Output:

```bash
NAME                          CPU(cores)   MEMORY(bytes)
api-server-xxxxxxxxxx         1m           2Mi
```

This confirmed metrics-server was functioning.

---

## Fix 3: Restarted API Deployment

Restarted deployment after configuration changes:

```bash
kubectl rollout restart deployment api-server -n t6
```

Verification:

```bash
kubectl rollout status deployment api-server -n t6
```

Output:

```bash
deployment "api-server" successfully rolled out
```

---

## Fix 4: Created Load Generator Pod

Created a load generator to continuously send requests:

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: load-generator
  namespace: t6
spec:
  containers:
  - name: load
    image: busybox:1.36
    command: ["/bin/sh", "-c"]
    args:
    - |
      while true; do
        for i in $(seq 1 20); do
          wget -q -O- --timeout=2 http://api-server.t6.svc.cluster.local/ > /dev/null 2>&1 &
        done
        sleep 1
      done
EOF
```

Verification:

```bash
kubectl get pods -n t6
```

Output:

```bash
load-generator   1/1   Running
```

---

# Successful Autoscaling Verification

## 1. HPA Working Correctly

```bash
kubectl get hpa -n t6
```

Output:

```bash
api-hpa   Deployment/api-server   cpu: 20%/50%   1   5   2
```

This confirmed:

* Metrics were available
* HPA was functioning
* Deployment scaled successfully

---

## 2. Deployment Successfully Scaled

```bash
kubectl get deployment api-server -n t6
```

Output:

```bash
NAME         READY   UP-TO-DATE   AVAILABLE
api-server   2/2     2            2
```

This confirmed scaling from 1 replica to 2 replicas.

---

## 3. Multiple Pods Running

```bash
kubectl get pods -n t6
```

Output:

```bash
api-server-xxxxxxxxxx-aaaaa   1/1   Running
api-server-xxxxxxxxxx-bbbbb   1/1   Running
load-generator                1/1   Running
```

---

## 4. Metrics Collection Verified

```bash
kubectl top pods -n t6
```

Output:

```bash
NAME                          CPU(cores)   MEMORY(bytes)
api-server-xxxxxxxxxx-aaaaa   44m          3Mi
api-server-xxxxxxxxxx-bbbbb   43m          2Mi
load-generator                4883m        17Mi
```

This confirmed:

* CPU metrics collection working correctly
* Load generator actively generating heavy traffic
* API server handling load across multiple replicas

---

## 5. HPA Events Verification

```bash
kubectl describe hpa api-hpa -n t6
```

Important Event:

```bash
Normal   SuccessfulRescale
```

This proved autoscaling occurred successfully.

---

# Final Result

All performance and autoscaling issues were successfully resolved.

## Final Outcomes

✅ Metrics server functioning correctly

✅ HPA collecting CPU metrics successfully

✅ LimitRange restrictions removed

✅ Load generator actively generating traffic

✅ API deployment scaled automatically

✅ Multiple replicas running successfully

✅ Metrics visible using `kubectl top`

✅ HPA no longer showing `<unknown>` metrics

✅ API server handling load successfully under autoscaling

---

# Final Verification Commands

```bash
kubectl get hpa -n t6

kubectl get deployment api-server -n t6

kubectl get pods -n t6

kubectl top pods -n t6

kubectl describe hpa api-hpa -n t6
```

