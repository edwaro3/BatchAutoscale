# Controlling Scale-Down Speed

This document describes the available levers for ensuring that scale-down is slower than scale-up in the Azure Batch autoscale system.

---

## Current behaviour

| Direction | Interval | Mechanism |
|-----------|----------|-----------|
| **Scale-up** | Every **5 minutes** | The orchestrator calls `EnableAutoScaleAsync` on each incoming request, gated by a 5-minute cooldown via `_scaleLockStore` |
| **Scale-down** | Every **15 minutes** | The pool's own `autoScaleEvaluationInterval` — Azure Batch self-evaluates the formula on this cadence |

Scale-down is already slower than scale-up, but the gap can be widened further using the levers below.

---

## Lever 1: Increase the pool evaluation interval

Change `TimeSpan.FromMinutes(15)` in the `EnableAutoScaleAsync` call to a higher value (maximum is 36 hours). For example, 30 minutes means the pool only re-evaluates scale-down every half hour:

```csharp
autoScaleEvaluationInterval: TimeSpan.FromMinutes(30)
```

**Effect**: The formula runs less often, so the pool takes longer to notice that demand has dropped.

**Limitation**: This only controls how often the formula runs — if the formula itself reacts to the latest sample, it can still scale down aggressively when it does run.

---

## Lever 2: Use time-averaged sampling in the formula template

Instead of reacting to the current task count, average over a longer window. In the Key Vault formula template:

```
// Average pending tasks over the last 30 minutes instead of using the latest sample
$samples = $ActiveTasks.GetSamplePercent(TimeInterval_Minute * 30);
$tasks = $samples < 70 ? max(0, $ActiveTasks.GetSample(1)) : avg($ActiveTasks.GetSample(TimeInterval_Minute * 30));
$TargetDedicatedNodes = max({1} / $tasksPerNode, min($tasks / $tasksPerNode, {0}));
```

**Effect**: The pool won't scale down until the average task count over 30 minutes drops — temporary dips in demand are smoothed out. The `$samples < 70` check falls back to the latest sample if insufficient history is available (e.g. shortly after the pool starts).

**This is the most effective lever** because it directly controls how quickly the formula reacts to falling demand, regardless of how often it evaluates.

---

## Lever 3: Set the node deallocation option

Ensure nodes finish their current tasks before being removed:

```
$NodeDeallocationOption = taskcompletion;
```

**Effect**: When the formula decides to scale down, nodes are not immediately deallocated. Each node waits until its running task completes before being removed from the pool.

**Limitation**: This doesn't slow down the *decision* to scale down — it only prevents nodes from being killed mid-task. Idle nodes are still removed immediately.

---

## Recommendation

**Combine lever 2 (time-averaged sampling) with the existing lever 1 (evaluation interval).**

| Setting | Value | Purpose |
|---------|-------|---------|
| Orchestrator cooldown | 5 minutes | Fast response to new demand (scale-up) |
| `autoScaleEvaluationInterval` | 15–30 minutes | Controls how often the pool self-evaluates (scale-down cadence) |
| Formula sample window | 30 minutes | Smooths out temporary demand dips before scaling down |
| `$NodeDeallocationOption` | `taskcompletion` | Prevents killing nodes mid-task |

This gives:
- **Fast scale-up**: The orchestrator pushes a new formula every 5 minutes with current task counts, reacting quickly to incoming demand.
- **Conservative scale-down**: The pool self-evaluates less frequently (every 15–30 minutes), using a 30-minute average — so it only scales down after sustained low demand, avoiding unnecessary cold starts.
