# Learning Kubernetes Kueue

This repository is a local development laboratory for learning [Kubernetes Kueue](https://kueue.sigs.k8s.io/), a job-level admission controller.

## 🛠 Development Environment

The environment is built on macOS (Darwin) using the following toolchain:

- **Docker Desktop**: v29.2.1
- **Kind**: v0.31.0
- **Helm**: v4.1.1
- **Kubernetes**: v1.31.0 (via Kind)

### Cluster Setup
The cluster is provisioned using a multi-node configuration to support specialized resource pools (Spot/On-Demand).

**kind-config.yaml:**
```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
- role: worker # Node 1: Generic
- role: worker # Node 2: Spot
- role: worker # Node 3: On-Demand
```

```bash
kind create cluster --name kueue-lab --config kind-config.yaml --image kindest/node:v1.31.0
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
- **`part-2/`**: "Smart Selection" - Multi-flavor fallback and automatic injection.
  - `resource-flavors.yaml`: Defines `spot` and `on-demand` flavors with taints/tolerations.
  - `cluster-queue.yaml`: A prioritized queue that falls back to On-Demand if Spot is full.
  - `local-queue.yaml`: Connects to the smart queue.
  - `sample-job.yaml`: A job template for testing the fallback logic.

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

## 🚀 Advanced Concepts (Part 2)

Part 2 explores **Smart Resource Selection** using multiple flavors and taints:

- **Flavor Prioritization**: By listing `spot` before `on-demand` in the `ClusterQueue`, Kueue always attempts to fit jobs into the "cheaper" pool first.
- **Automatic Injection**: Kueue automatically adds `nodeSelector` and `tolerations` to the Job based on the selected flavor. This means users don't need to know about the underlying infrastructure details.
- **Tainted Node Protection**: Using `NoSchedule` taints on Spot/On-Demand nodes ensures that ONLY Kueue-managed jobs (which get the injected tolerations) can land on these specialized resources.

### Execution Plan
1. **Multi-Node Cluster**: A Kind cluster with 3 worker nodes (Generic, Spot, On-Demand).
2. **Specialized Taints**: One node is tainted as `spot`, another as `on-demand`.
3. **Infrastructure**: Apply ResourceFlavors that match these taints.
4. **Fallback Test**:
   - Create **Job 1**: Fits in the 500m Spot quota. Kueue injects `spot` tolerations.
   - Create **Job 2**: Spot quota is now exhausted (only 100m left). Kueue automatically selects the `on-demand` flavor and injects the corresponding tolerations.

## 🧪 Verification Commands

To verify the installation and API versions:

```bash
# Check Kueue pods
kubectl get pods -n kueue-system

# Verify CRD Storage Version (should be v1beta2)
kubectl get crd clusterqueues.kueue.x-k8s.io -o jsonpath='{range .spec.versions[*]}{.name}{" (Storage: "}{.storage}{")
"}{end}'
```
