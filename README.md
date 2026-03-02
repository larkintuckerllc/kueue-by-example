# Learning Kubernetes Kueue

This repository is a local development laboratory for learning [Kubernetes Kueue](https://kueue.sigs.k8s.io/), a job-level admission controller.

## 🛠 Development Environment

The environment is built on macOS (Darwin) using the following toolchain:

- **Docker Desktop**: v29.2.1
- **Kind**: v0.31.0
- **Helm**: v4.1.1
- **Kubernetes**: v1.31.0 (via Kind)

### Cluster Setup
The cluster was provisioned using `kindest/node:v1.31.0` to ensure stability for modern batch features like Pod Scheduling Readiness (Pod Gates) and advanced Job policies.

```bash
kind create cluster --image kindest/node:v1.31.0 --name kueue-lab
```

## 🚀 Kueue Installation

Kueue was installed using the official **Helm OCI chart** to ensure the most modern installation path.

- **Version**: `v0.16.2` (Current Stable)
- **Namespace**: `kueue-system`
- **API Version**: `v1beta2` (Storage Version)

### Installation Command
```bash
helm install kueue oci://registry.k8s.io/kueue/charts/kueue 
  --version 0.16.2 
  --namespace kueue-system 
  --create-namespace 
  --wait
```

## 📂 Project Structure

- **`part-1/`**: "Hello World" - Introduction to basic Admission Control.
  - `resource-flavor.yaml`: Defines the hardware/resource type.
  - `cluster-queue.yaml`: The cluster-wide resource pool (1 CPU quota).
  - `local-queue.yaml`: The namespace-specific entry point.
  - `sample-job.yaml`: A basic Job with 3 pods (100m CPU each).

## 🧠 Core Concepts (Part 1)

The Part 1 example illustrates the fundamental **Admission Control** workflow:

- **ResourceFlavor**: The **"What"** (Currency). Defines a hardware or resource type (e.g., standard nodes, GPU nodes, Spot instances). Even a simple setup requires at least one flavor.
- **ClusterQueue**: The **"How Much"** (Bank Vault). A cluster-wide pool that holds the total available quota for specific resource flavors.
- **LocalQueue**: The **"Entry Point"** (Debit Card). A namespace-scoped object that connects users in that namespace to a `ClusterQueue`.
- **Job Admission**: The **"Gatekeeper"**.
  1. Jobs are submitted with `suspend: true`.
  2. Kueue intercepts the Job based on the `kueue.x-k8s.io/queue-name` label.
  3. Kueue verifies available quota in the `ClusterQueue`.
  4. Once admitted, Kueue flips `suspend: false`, and the standard K8s scheduler starts the pods.

## 🧪 Verification Commands

To verify the installation and API versions:

```bash
# Check Kueue pods
kubectl get pods -n kueue-system

# Verify CRD Storage Version (should be v1beta2)
kubectl get crd clusterqueues.kueue.x-k8s.io -o jsonpath='{range .spec.versions[*]}{.name}{" (Storage: "}{.storage}{")
"}{end}'
```
