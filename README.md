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

## 🧪 Verification Commands

To verify the installation and API versions:

```bash
# Check Kueue pods
kubectl get pods -n kueue-system

# Verify CRD Storage Version (should be v1beta2)
kubectl get crd clusterqueues.kueue.x-k8s.io -o jsonpath='{range .spec.versions[*]}{.name}{" (Storage: "}{.storage}{")
"}{end}'
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
- **`part-3/`**: "Borrowing" - Resource sharing via Cohorts.
  - `resource-flavor.yaml`: A shared default resource flavor.
  - `cluster-queues.yaml`: Two ClusterQueues in the same `cohortName`.
  - `local-queues.yaml`: Entry points for `team-a` and `team-b` namespaces.
  - `borrow-job.yaml`: A job for Team A that exceeds its nominal quota.
  - `team-b-job.yaml`: A job for Team B to test cohort-level queuing.
- **`part-4/`**: "Preemption" - SLA management via PriorityClasses.
  - `priority-classes.yaml`: Defines `low-priority` and `high-priority` workload classes.
  - `infrastructure.yaml`: A ClusterQueue configured for preemption.
  - `background-job.yaml`: A long-running, low-priority workload.
  - `critical-job.yaml`: A high-priority job that triggers the eviction of background tasks.

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

## 🤝 Resource Sharing & Borrowing (Part 3)

Part 3 demonstrates how Kueue solves the "stranded resources" problem using **Cohorts**:

- **Cohorts (`cohortName`)**: A logical grouping of multiple `ClusterQueues`. Members of the same cohort can borrow unused resources from each other's nominal quotas.
- **Borrowing**: If a team submits a job that exceeds their own quota, Kueue checks if other members of the cohort are using their quotas. If they are idle, the team can "borrow" from them.
- **Cross-Namespace Queuing**: If the *entire* cohort's capacity is full, new jobs are queued globally, regardless of which namespace they were submitted to.

### Execution Plan
1. **Multi-Namespace Setup**: Separate `team-a` and `team-b` namespaces.
2. **Shared Pool**: Create two `ClusterQueues` (each with 500m CPU) and join them into a cohort named `shared-pool`.
3. **The Borrow Test**:
   - Submit **Job 1 (Team A)** for **800m CPU**.
   - **Result**: Admitted immediately. It uses its own 500m and **borrows 300m** from Team B's idle quota.
4. **The Queuing Test**:
   - While Job 1 is running, submit **Job 2 (Team B)** for **300m CPU**.
   - **Result**: PENDING. Total cohort capacity is 1000m, but 800m is already in use. Team B must wait for Team A to finish and return the borrowed resources.

## ⚡️ Priority & Preemption (Part 4)

Part 4 demonstrates how Kueue ensures SLAs for critical workloads through **PriorityClasses** and **Preemption**:

- **WorkloadPriorityClass**: Defines a numeric priority for different types of jobs (e.g., 100 for Low, 1000 for High).
- **Within-Queue Preemption**: When configured with `withinClusterQueue: LowerPriority`, Kueue will evict already-running lower-priority jobs to make room for a higher-priority one if the quota is full.
- **Automatic Eviction**: Kueue handles the systematic termination of the evicted job's pods and returns it to the queue, ensuring the high-priority job can start immediately.

### Execution Plan
1. **Priority Definitions**: Create `low-priority` and `high-priority` WorkloadPriorityClasses.
2. **Preemption Policy**: Configure a `ClusterQueue` with a 1000m CPU quota and enable `withinClusterQueue` preemption.
3. **The Fill-The-Pool Test**:
   - Submit **Background Job** (Low Priority) for **800m CPU**.
   - **Result**: Admitted and starts running immediately.
4. **The Preemption Test**:
   - Submit **Critical Job** (High Priority) for **400m CPU**.
   - **Result**: Even though only 200m CPU is available, the Critical Job is admitted. Kueue **evicts** the Background Job, terminates its pods, and starts the Critical Job pods immediately.
