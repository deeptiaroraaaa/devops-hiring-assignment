# Challenge 2 – Troubleshooting Report

## Problem Faced

The deployment in namespace `t2` was not reaching the desired replica count. The ResourceQuota prevented additional pods from being created.

### Observed Output

```bash
kubectl get resourcequota -A
```

```text
NAMESPACE   NAME          AGE   REQUEST                                   LIMIT
t2          tight-quota   8h    pods: 2/2, requests.memory: 128Mi/150Mi
```

```bash
kubectl describe resourcequota tight-quota -n t2
```

```text
Resource         Used   Hard
pods             2      2
requests.memory  128Mi  150Mi
```

The deployment status showed:

```bash
kubectl get deploy -n t2
```

```text
NAME     READY   UP-TO-DATE   AVAILABLE   AGE
task-2   2/3     0            2           8h
```

---

## Root Cause

The namespace had a strict `ResourceQuota`:

* Maximum allowed pods: `2`
* Maximum memory requests: `150Mi`

During testing, the deployment replica count and memory usage were increased, which exceeded the quota limits. Kubernetes therefore blocked creation of additional pods.

---

## Fix Applied

The deployment replica count was reduced back to the allowed limit.

### Command Used

```bash
kubectl scale deploy task-2 -n t2 --replicas=2
```

### Verification

```bash
kubectl get deploy -n t2
kubectl get pods -n t2
```

Result:

```text
NAME     READY   UP-TO-DATE   AVAILABLE   AGE
task-2   2/2     0            2           8h
```

```text
NAME                      READY   STATUS    RESTARTS   AGE
task-2-75cd798754-4tc5x   1/1     Running   0          11m
task-2-75cd798754-gv5g8   1/1     Running   0          11m
```

---

## Final Outcome

The deployment successfully returned to a healthy state after reducing replicas within the namespace quota limits. All pods became stable and the challenge was completed successfully.
