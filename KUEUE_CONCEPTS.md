# A Journey Through Kubernetes Kueue

Kubernetes was originally designed for long-running services (web servers, databases) that need to be "always on." When it comes to Batch Jobs (AI/ML training, rendering, simulations), the default Kubernetes scheduler has a significant limitation: it tries to schedule everything immediately.

If you submit 1,000 jobs to a cluster that can only run 10, the scheduler creates 1,000 sets of pods. They all fight for resources, the control plane gets overwhelmed, and the cluster can become unstable.

**Kueue** solves this by acting as a **Job-Level Admission Controller**. It doesn't schedule pods; it decides *when* a Job object is allowed to even ask for pods.

---

## Chapter 1: The Gatekeeper (Admission Control)
*Reference: `part-1/`*

### The Problem
Imagine a bank with 5 tellers. If 500 customers rush in at once and crowd the counter, nobody gets served efficiently. You need a rope line and a security guard to let people in only when a teller is free.

### The Solution: ClusterQueue
In our first example, we created a **ClusterQueue**—our "bank vault"—with a strict limit (e.g., 1 CPU).

When we submitted the `sample-job.yaml`, Kueue intercepted it. It saw the job had `suspend: true`. Instead of letting the job create pods immediately, Kueue calculated the cost.
- **If the vault has funds (Quota):** Kueue "admits" the job by setting `suspend: false`. The job runs.
- **If the vault is empty:** The job stays suspended. It sits in a queue, ordered by FIFO (First-In-First-Out).

**Key Takeaway:** Kueue protects the cluster stability by queuing Job objects *before* they create Pods.

---

## Chapter 2: The Economist (Smart Selection)
*Reference: `part-2/`*

### The Problem
Not all compute is created equal. Cloud providers offer "Spot" instances (unreliable but 90% cheaper) and "On-Demand" instances (reliable but expensive).
A cost-conscious engineer wants to run jobs on Spot instances whenever possible but switch to On-Demand if Spot is sold out. Hard-coding `nodeSelector: spot` into every job script is brittle and frustrating for users.

### The Solution: ResourceFlavors & Fallback
In Part 2, we defined two **ResourceFlavors**: `spot` and `on-demand`.
We configured the **ClusterQueue** with an ordered list:
1. `spot`
2. `on-demand`

When a job arrives:
1. Kueue checks the `spot` quota. If available, it assigns the job to `spot` and **automatically injects** the specific tolerations and node selectors needed to land on Spot nodes.
2. If `spot` is full, Kueue automatically tries the next flavor, `on-demand`.

**Key Takeaway:** Kueue abstracts the infrastructure. Users just ask for "cpu", and Kueue finds the best (cheapest) place to put it based on policy.

---

## Chapter 3: The Cooperative (Cohorts)
*Reference: `part-3/`*

### The Problem
Organizations often split clusters into strict silos. "Team A gets 50 CPUs, Team B gets 50 CPUs."
But what happens on Tuesday when Team A is on vacation (0 usage) and Team B has a deadline (needs 80 CPUs)?
In a rigid system, Team A's resources sit idle ("stranded capacity") while Team B's jobs sit pending. This is wasteful.

### The Solution: Cohorts
In Part 3, we introduced **Cohorts**. We took Team A's queue and Team B's queue and gave them the same label: `cohort: shared-pool`.

1. **Guaranteed Quota:** Team A still owns their 50 CPUs. If they want them, they get them.
2. **Borrowing:** If Team A is not using their quota, the resources go into a "shared pot." Team B can now borrow those idle CPUs.
3. **Reclaiming:** If Team A suddenly submits a job, Team B stops borrowing (or queues), ensuring fairness.

**Key Takeaway:** Cohorts maximize cluster utilization by turning rigid quotas into flexible, shared pools, while still protecting ownership.

---

## Chapter 4: The Executive (Priority & Preemption)
*Reference: `part-4/`*

### The Problem
Your cluster is running at 100% efficiency thanks to Cohorts. It's full of "background" jobs—data crunching that can happen anytime.
Suddenly, a "Production Critical" bug fix job arrives. It needs resources *now*.
In a standard FIFO queue, the critical job would have to wait for hours behind the background jobs.

### The Solution: Preemption
In Part 4, we gave the cluster a "conscience" using **WorkloadPriorityClass**.
- Background Jobs = Priority 100
- Critical Jobs = Priority 1000

We configured the ClusterQueue with `preemption.withinClusterQueue: LowerPriority`.

When the Critical Job arrived and found the cluster full:
1. Kueue identified that the running jobs had lower priority.
2. Kueue **evicted** (stopped) the background jobs.
3. The resources were freed instantly.
4. The Critical Job started.

**Key Takeaway:** Preemption ensures that high-value work never waits, allowing you to safely fill your cluster with low-priority work to maximize ROI, knowing you can clear the deck instantly if needed.
