# lau-scheduling-theory

**Scheduling theory in Rust** — optimal resource allocation over time: single-machine and parallel scheduling, priority dispatching rules, Johnson's algorithm, branch-and-bound optimization, preemptive scheduling, due-date objectives, precedence constraints, resource-constrained scheduling, shop scheduling, and multi-agent fleet coordination.

## What This Does

This crate implements the core algorithms from classical scheduling theory:

- **Job model** — jobs with processing times, weights, due dates, release dates, and deadlines; schedules with computed objectives (makespan, total completion time, weighted completion time, max lateness, total tardiness)
- **Priority rules** — SPT, EDD, WSPT, LPT, FCFS dispatching heuristics
- **Single-machine scheduling** — schedule jobs on one machine using any priority rule
- **Parallel-machine scheduling** — identical parallel machines with list scheduling and makespan lower bounds
- **Flow shop** — Johnson's algorithm for the optimal 2-machine flow shop (F2 ‖ C_max)
- **Branch and bound** — exact optimization for total weighted completion time (single machine) and makespan (parallel machines)
- **Preemptive scheduling** — Shortest Remaining Processing Time (SRPT) and preemptive EDD
- **Due-date objectives** — minimize L_max (EDD, optimal), minimize ΣT_j (slack heuristic), minimize number of tardy jobs (Moore's algorithm)
- **Precedence constraints** — topological sorting, SPT-with-precedence scheduling
- **Resource-constrained scheduling** — serial schedule generation scheme with resource capacity tracking
- **Shop scheduling** — open shop (greedy/LPT dispatching) and job shop (greedy route-based dispatching)
- **Agent fleet scheduling** — multi-agent task assignment with specializations, dependencies, capacity scaling, utilization and load-balance metrics

## Key Idea

Scheduling problems have the structure: given *n* jobs and *m* machines, find an assignment and ordering that optimizes an objective (makespan, total completion time, tardiness, etc.). This library provides both **exact** methods (branch-and-bound for small instances) and **heuristic** methods (priority rules, list scheduling, Johnson's algorithm) for the standard problem variants.

The `Schedule` type tracks start/end times per job and computes all standard objectives via methods like `.makespan()`, `.total_tardiness(jobs)`, `.max_lateness(jobs)`.

## Install

```toml
[dependencies]
lau-scheduling-theory = "0.1.0"
```

Requires Rust 2021 edition. Depends on `serde` (serialization) and `nalgebra` (linear algebra).

## Quick Start

### Single Machine with Priority Rules

```rust
use lau_scheduling_theory::*;

let jobs = vec![
    Job::new(0, 5.0).with_due_date(10.0),
    Job::new(1, 3.0).with_due_date(7.0).with_weight(2.0),
    Job::new(2, 8.0).with_due_date(20.0),
    Job::new(3, 2.0).with_due_date(5.0),
];

// SPT minimizes total completion time
let schedule = SingleMachineScheduler::schedule(&jobs, PriorityRule::SPT);
println!("Makespan: {}", schedule.makespan());
println!("Total completion time: {}", schedule.total_completion_time());

// EDD minimizes maximum lateness (optimal)
let edd_schedule = SingleMachineScheduler::schedule(&jobs, PriorityRule::EDD);
println!("Max lateness: {}", edd_schedule.max_lateness(&jobs));
```

### Johnson's Algorithm (2-Machine Flow Shop)

```rust
use lau_scheduling_theory::flow_shop::{JohnsonAlgorithm, FlowShopJob};

let jobs = vec![
    FlowShopJob { id: 0, time_a: 3.0, time_b: 6.0 },
    FlowShopJob { id: 1, time_a: 8.0, time_b: 2.0 },
    FlowShopJob { id: 2, time_a: 4.0, time_b: 5.0 },
    FlowShopJob { id: 3, time_a: 9.0, time_b: 1.0 },
];
let optimal_seq = JohnsonAlgorithm::sequence(&jobs);
let schedule = JohnsonAlgorithm::schedule(&jobs);
println!("Optimal makespan: {}", schedule.makespan());
```

### Branch and Bound (Exact Optimization)

```rust
use lau_scheduling_theory::branch_and_bound::BranchAndBound;

let jobs = vec![
    Job::new(0, 4.0).with_weight(1.0),
    Job::new(1, 3.0).with_weight(2.0),
    Job::new(2, 5.0).with_weight(1.0),
    Job::new(3, 2.0).with_weight(3.0),
];

// Find the sequence minimizing total weighted completion time
let optimal = BranchAndBound::optimize_twct(&jobs);
let schedule = BranchAndBound::schedule_from_sequence(&jobs, &optimal);
println!("Optimal TWCT: {}", schedule.total_weighted_completion_time(&jobs));
```

### Parallel Machines

```rust
use lau_scheduling_theory::parallel_machine::ParallelMachineScheduler;

let m = 3; // 3 identical machines
let lb = ParallelMachineScheduler::makespan_lower_bound(&jobs, m);
let schedule = ParallelMachineScheduler::schedule_lpt(&jobs, m);
println!("Lower bound: {}, LPT makespan: {}", lb, schedule.makespan());
```

### Minimize Number of Tardy Jobs (Moore's Algorithm)

```rust
use lau_scheduling_theory::due_date::DueDateScheduler;

let moore_schedule = DueDateScheduler::minimize_num_tardy(&jobs);
let num_tardy = DueDateScheduler::num_tardy(&jobs, &moore_schedule);
println!("Tardy jobs: {}", num_tardy);
```

### Agent Fleet Scheduling

```rust
use lau_scheduling_theory::agent_fleet::{AgentFleetScheduler, Agent, AgentTask};

let agents = vec![
    Agent { id: 0, capacity: 1.0, specializations: vec!["compute".into()] },
    Agent { id: 1, capacity: 2.0, specializations: vec!["compute".into(), "io".into()] },
];
let tasks = vec![
    AgentTask { id: 0, processing_time: 5.0, task_type: "compute".into(), priority: 1.0, deadline: Some(10.0), dependencies: vec![] },
    AgentTask { id: 1, processing_time: 3.0, task_type: "io".into(), priority: 2.0, deadline: None, dependencies: vec![0] },
];
let schedule = AgentFleetScheduler::schedule(&tasks, &agents);
let util = AgentFleetScheduler::agent_utilization(&schedule);
println!("Utilization: {:?}", util);
```

## API Reference

### `job` — Core Types

| Type | Description |
|------|-------------|
| `Job` | A job with id, processing time, weight, due/release date, deadline |
| `ScheduledJob` | A placed job with start/end time and machine assignment |
| `Schedule` | Complete schedule across one or more machines |
| `.makespan()` | C_max = max end time |
| `.total_completion_time()` | Σ C_j |
| `.total_weighted_completion_time(jobs)` | Σ w_j C_j |
| `.max_lateness(jobs)` | L_max = max(C_j − d_j) |
| `.total_tardiness(jobs)` | Σ max(C_j − d_j, 0) |

### `priority` — Dispatching Rules

| Rule | Optimality |
|------|-----------|
| `SPT` | Minimizes Σ C_j on single machine (optimal) |
| `EDD` | Minimizes L_max on single machine (optimal) |
| `WSPT` | Minimizes Σ w_j C_j on single machine (optimal) |
| `LPT` | Heuristic for parallel machine makespan |
| `FCFS` | First-come-first-served (by job id) |

### `single_machine` — 1‖objective

| Fn | Description |
|----|-------------|
| `SingleMachineScheduler::schedule(jobs, rule)` | Schedule using any priority rule |

### `parallel_machine` — P_m‖objective

| Fn | Description |
|----|-------------|
| `ParallelMachineScheduler::schedule(jobs, m, rule)` | List scheduling on m machines |
| `.schedule_lpt(jobs, m)` | LPT heuristic for makespan |
| `.makespan_lower_bound(jobs, m)` | max(max p_j, Σp_j / m) |

### `flow_shop` — F2‖C_max

| Fn | Description |
|----|-------------|
| `JohnsonAlgorithm::sequence(jobs)` | Optimal sequence for 2-machine flow shop |
| `.schedule(jobs)` | Full schedule with start/end times |
| `.makespan(jobs, sequence)` | Evaluate makespan of any sequence |

### `branch_and_bound` — Exact Methods

| Fn | Description |
|----|-------------|
| `BranchAndBound::optimize_twct(jobs)` | Optimal sequence for Σ w_j C_j |
| `.optimize_makespan_parallel(jobs, m)` | Optimal machine assignment for C_max |
| `.schedule_from_sequence(jobs, seq)` | Build schedule from an ordered sequence |

### `preemptive` — Preemption Allowed

| Fn | Description |
|----|-------------|
| `PreemptiveScheduler::srpt(jobs)` | Shortest Remaining Processing Time (segments) |
| `.preemptive_edd(jobs)` | Preemptive EDD for L_max |
| `.total_completion_time(jobs, segments)` | Compute Σ C_j from segments |

### `due_date` — Due-Date Objectives

| Fn | Description |
|----|-------------|
| `DueDateScheduler::minimize_max_lateness(jobs)` | EDD schedule (optimal for L_max) |
| `.minimize_total_tardiness(jobs)` | Slack heuristic for Σ T_j |
| `.minimize_num_tardy(jobs)` | Moore's algorithm for Σ U_j (optimal) |
| `.num_tardy(jobs, schedule)` / `.max_lateness(...)` / `.total_tardiness(...)` | Evaluate objectives |

### `precedence` — Precedence Constraints

| Fn | Description |
|----|-------------|
| `PrecedenceScheduler::topological_sort(n, constraints)` | Valid ordering via Kahn's algorithm |
| `.schedule(jobs, constraints)` | Schedule respecting precedence |
| `.schedule_spt_with_precedence(jobs, constraints)` | SPT within topological order |
| `.is_valid_sequence(seq, constraints)` | Check a sequence |

### `resource_constrained` — RCPSP

| Fn | Description |
|----|-------------|
| `ResourceConstrainedScheduler::schedule(jobs, resources, precedence)` | Serial SGS with resource tracking |

### `shop` — Open Shop and Job Shop

| Fn | Description |
|----|-------------|
| `OpenShopScheduler::schedule(tasks, m)` | Greedy LPT dispatching for open shop |
| `JobShopScheduler::schedule(jobs, m)` | Greedy dispatching for job shop (fixed routes) |

### `agent_fleet` — Multi-Agent Task Assignment

| Type / Fn | Description |
|-----------|-------------|
| `Agent` | Agent with capacity (speed multiplier) and specializations |
| `AgentTask` | Task with type, priority, deadline, dependencies |
| `AgentFleetScheduler::schedule(tasks, agents)` | Greedy assignment respecting specializations & deps |
| `.agent_utilization(schedule)` | Fraction of makespan each agent is busy |
| `.load_balance(schedule)` | Std dev of agent busy times |

## How It Works

1. **Priority rules sort by a single key.** SPT sorts by p_j ascending, EDD by d_j ascending, WSPT by p_j/w_j ascending. These are provably optimal for their respective objectives on a single machine.

2. **Parallel machines use list scheduling.** Jobs are sorted by the priority rule, then each job is assigned to the least-loaded machine. LPT (longest first) gives a 4/3 − 1/(3m) approximation for makespan.

3. **Johnson's algorithm partitions jobs.** Jobs with p_a ≤ p_b go to the front (sorted by p_a ascending); jobs with p_a > p_b go to the back (sorted by p_b descending). This yields the optimal sequence for F2 ‖ C_max.

4. **Branch and bound prunes with WSPT lower bounds.** At each partial sequence, the remaining jobs are relaxed to WSPT order. If the lower bound exceeds the best known solution, the subtree is pruned. Candidates are explored in WSPT order for faster convergence.

5. **Moore's algorithm greedily drops jobs.** Process in EDD order; when a tardy job appears, drop the longest one seen so far. This maximizes the number of on-time jobs optimally.

6. **Agent fleet uses multi-pass greedy assignment.** Tasks are sorted by priority/deadline, then iteratively assigned to the least-loaded compatible agent. Multiple passes handle dependencies.

## The Math

**1‖Σ C_j (SPT):** The SPT rule is optimal for minimizing total completion time on a single machine. Proof by pairwise exchange: if a longer job precedes a shorter one, swapping them reduces the sum.

**1‖L_max (EDD):** The EDD (Earliest Due Date) rule is optimal for minimizing maximum lateness. Proof by contradiction using Jackson's rule.

**1‖Σ w_j C_j (WSPT):** Weighted SPT (sort by p_j / w_j) is optimal for minimizing total weighted completion time. Smith's rule, 1956.

**F2‖C_max (Johnson):** For two-machine flow shop, Johnson's algorithm produces the optimal sequence in O(n log n). Partition: J₁ = {j : p₁ⱼ ≤ p₂ⱼ}, J₂ = {j : p₁ⱼ > p₂ⱼ}. Sequence J₁ by p₁ⱼ ascending, then J₂ by p₂ⱼ descending.

**P_m‖C_max (LPT):** The LPT (Longest Processing Time first) list scheduling heuristic gives C_max ≤ (4/3 − 1/(3m)) × C*_max, where m is the number of machines.

**1‖Σ U_j (Moore):** Moore-Hodgson algorithm: process in EDD order, maintain a set of on-time jobs. When a tardy job would appear, remove the longest job in the set. This is optimal for minimizing the number of tardy jobs.

**SRPT:** Shortest Remaining Processing Time is optimal for minimizing Σ C_j with preemption on a single machine (and also for mean flow time in the presence of release dates).

## Test Suite

59 integration tests covering:
- Job construction and schedule objective computation (makespan, TWCT, max lateness, total tardiness)
- All five priority rules
- Single-machine scheduling with each rule
- Parallel-machine list scheduling and makespan lower bounds
- Johnson's algorithm optimality and makespan computation
- Branch and bound for TWCT (optimality verification)
- Branch and bound for parallel makespan
- SRPT preemptive scheduling
- EDD optimality for L_max
- Moore's algorithm for number of tardy jobs
- Precedence constraint topological sort and schedule validity
- Resource-constrained scheduling
- Open shop and job shop scheduling
- Agent fleet scheduling with specializations, dependencies, utilization

## License

MIT
