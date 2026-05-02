# Short Notes: Chapter 1 - Reliable, Scalable, and Maintainable Applications

* **Data-Intensive Applications**: Apps where data complexity, volume, and movement (not CPU power) are the main challenges.

## 1. Reliability (Fault Tolerance)
* **Definition**: Continuing to work correctly even when components fail.
* **Fault vs. Failure**: 
  * **Fault**: One component breaks (e.g., a hard drive).
  * **Failure**: The entire system stops providing service.
* **Goal**: Prevent individual _faults_ from becoming system-wide _failures_ (often using redundancy).

## 2. Scalability
* **Definition**: A system's ability to cope with increased load smoothly.
* **Load**: Need to define load parameters (e.g., Requests Per Second, concurrent users).
* **Performance**: As load increases, how does latency react?
  * **Percentiles matter**: Use p95, p99, p99.9 instead of averages. An average hides terrible experiences for outlier users (tail latency).
* **Vertical Scaling (Scale Up):** Bigger machine. Simple but has physical limits and is a single point of failure.
* **Horizontal Scaling (Scale Out):** More machines behind a Load Balancer. Near-infinite scale but requires distributed systems thinking.

## 3. Maintainability
* **Definition**: Creating a system that current and future engineers find easy to work with over its lifecycle.
* **Three Pillars:**
  1. **Operability**: Make the system easy to monitor and keep running smoothly (good telemetry and dashboards).
  2. **Simplicity**: Prevent accidental complexity. Use clean, understandable abstractions.
  3. **Evolvability**: Make it easy to modify boundaries and add features as business requirements change.

## Key Takeaways
* You can't just slap a "scalable" label on a system; you have to explain _how_ you handle _which_ kinds of load.
* Hardware failures, software bugs, and human errors are inevitable; design systems to recover from them.
* Code spends most of its lifecycle in maintenance. Optimizing for the next developer's sanity is financially critical.

## Fault Types (Kleppmann's 3 Categories)
| Fault Type | Example | Fix |
|---|---|---|
| Hardware | Disk crash, power outage | RAID, redundant power, replication |
| Software | Runaway process, cascading failure | Isolation, thorough testing, crash-restart |
| Human | Wrong config, deleted data | Staging env, rollbacks, gradual rollouts |

## Availability Nines (Quick Reference)
| Availability | Downtime/Year | Downtime/Month |
|---|---|---|
| 99% (2 nines) | ~3.65 days | ~7.2 hours |
| 99.9% (3 nines) | ~8.7 hours | ~43 minutes |
| 99.99% (4 nines) | ~52 minutes | ~4.3 minutes |
| 99.999% (5 nines) | ~5.3 minutes | ~26 seconds |

## SLI / SLO / SLA (Mastery-Level)
* **SLI** (Indicator): The raw metric tracked (e.g., % of HTTP 200 responses).
* **SLO** (Objective): Internal engineering target (e.g., 99.9% requests < 200ms).
* **SLA** (Agreement): Legal/financial contract with customer based on the SLO.
