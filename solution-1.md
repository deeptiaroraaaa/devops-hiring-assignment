# Challenge 1 – Deploy the KIND Cluster

## Objective

Deploy the Terraform stack successfully and create a working KIND Kubernetes cluster with:

* 1 control-plane node
* 3 worker nodes
* Working kubectl context: `kind-sanjay-challenge`

---

# Initial Issues Faced

While running:

```bash
terraform init
terraform apply
```

multiple errors occurred during cluster creation.

---

# Issue 1 – Incorrect KIND Configuration File Path

## Symptoms Observed

Terraform failed during the `kind_cluster` resource creation.

Error received:

```bash
ERROR: failed to create cluster: error reading file:
open /kubernetes/kind.config.yaml: no such file or directory
```

Because of this, the Kubernetes context was also not created:

```bash
error: no context exists with the name: "kind-sanjay-challenge"
```

---

# Investigation

I inspected the Terraform file responsible for cluster creation.

Command used:

```bash
cat main.tf
```

Inside the `local-exec` provisioner, the following command was present:

```bash
kind create cluster --name ${var.cluster_name} --config /kubernetes/kind.config.yaml
```

The file path was incorrect.

The actual file name inside the repository was:

```bash
kind-config.yaml
```

not:

```bash
kind.config.yaml
```

Additionally, the path needed to reference the parent directory correctly.

---

# Root Cause

Incorrect KIND configuration file path inside Terraform:

* Wrong filename (`kind.config.yaml`)
* Wrong path reference

---

# Fix Applied

I updated the Terraform configuration from:

```bash
kind create cluster --name ${var.cluster_name} --config /kubernetes/kind.config.yaml
```

to:

```bash
kind create cluster --name ${var.cluster_name} --config ../kubernetes/kind-config.yaml
```

---

# Verification

After fixing the path, I re-ran:

```bash
terraform apply
```

The KIND cluster was created successfully.

Verification command:

```bash
kubectl --context kind-sanjay-challenge get nodes
```

Output confirmed:

* control-plane node running
* worker nodes created
* kubectl context working correctly

---

# Issue 2 – Missing Service Account in Namespace t5

## Symptoms Observed

Terraform failed during challenge setup while creating the `tls-client` pod.

Error received:

```bash
pods "tls-client" is forbidden:
serviceaccount "default" not found
```

---

# Investigation

I checked the namespace resources and confirmed the default service account was missing in namespace `t5`.

---

# Root Cause

The `default` service account was not available inside namespace `t5`, causing pod creation to fail.

---

# Fix Applied

I manually recreated the default service account:

```bash
kubectl create serviceaccount default -n t5
```

Then I re-ran:

```bash
terraform apply
```

---

# Verification

The `tls-client` pod was created successfully.

Verification command:

```bash
kubectl get pods -n t5
```

---

# Challenges Faced During Debugging

* Understanding Terraform `local-exec` provisioners
* Identifying relative path issues inside Terraform
* Understanding how KIND cluster creation works
* Debugging Kubernetes context creation failures
* Investigating missing service account errors
* Understanding namespace-level Kubernetes resources

---

# Tools Used

* Terraform
* kubectl
* KIND
* Docker
* Linux shell commands

---

# Final Result

Successfully deployed the local Kubernetes environment with:

* Working KIND cluster
* Working kubectl context
* Kubernetes system components running successfully
* Challenge workloads deployed successfully

Verification:

```bash
kubectl get nodes
kubectl get pods -A
```

All required cluster resources were successfully created.
