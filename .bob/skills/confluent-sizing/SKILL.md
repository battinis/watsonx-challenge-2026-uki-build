---
name: confluent-sizing
description: Use when the user wants to size a Confluent Kafka cluster or estimate Confluent Cloud costs — covers on-prem broker/partition/storage sizing and Confluent Cloud tier selection and monthly cost estimation.
---

# Confluent Sizing

Follow these steps to produce a complete Confluent sizing assessment covering both on-prem cluster sizing and Confluent Cloud cost estimation.

---

## Step 1 — Gather Workload Inputs

Use `ask_followup_question` to collect the following from the user. Ask for all of them in one message if possible; group related questions together.

**Traffic and message profile:**
- Peak message produce rate (messages/second or MB/s)
- Average message size (bytes or KB)
- Number of distinct topics
- Desired replication factor (default: 3)
- Expected consumer-to-producer fan-out ratio (e.g., 1× = one consumer group, 3× = three consumer groups reading the same data)

**Retention and storage:**
- Retention period per topic (hours or days)
- Whether tiered storage or infinite retention is needed

**Availability and reliability:**
- Target availability (e.g., 99.9%, 99.99%)
- Is multi-region or multi-AZ required?

**Deployment context:**
- On-prem / self-managed Kafka, Confluent Platform, or Confluent Cloud (or both)?
- If Confluent Cloud: preferred cloud provider (AWS / Azure / GCP) and region

If the user has already provided some of these, skip those questions.

---

## Step 2 — On-Prem / Confluent Platform Cluster Sizing

Compute the following sizing parameters from the gathered inputs. Show your working clearly.

### 2a. Partition Count

```
partitions_per_topic = max(produce_rate_MB_s / target_throughput_per_partition_MB_s, 1)
```

- Target throughput per partition: **10 MB/s** is a safe default for commodity hardware; use **20 MB/s** for high-performance SSDs.
- Round up to the nearest power of 2 or a round number for operational simplicity.
- Multiply by the number of topics to get total partition count.

### 2b. Broker Count

```
min_brokers = ceil(total_partitions / max_partitions_per_broker)
```

- Max partitions per broker: **4,000** (Confluent recommendation for stability and rebalance time).
- Add the replication factor overhead: effective data rate per broker = (produce_rate × replication_factor) / broker_count — ensure no broker exceeds **80% of NIC bandwidth**.
- Enforce a minimum of **3 brokers** for any replicated cluster; 6 or more for 99.99% targets.
- Add 1 extra broker as a rolling-maintenance spare for clusters with SLA requirements.

### 2c. Storage per Broker

```
total_raw_storage_GB = produce_rate_MB_s × retention_seconds × replication_factor / 1024
storage_per_broker_GB = total_raw_storage_GB / broker_count
```

- Add **20% headroom** for indexes, compaction overhead, and OS.
- For tiered storage (Confluent Platform 6.0+ / Confluent Cloud): size local disk for only the **hot tier** (typically 1–7 days); remainder offloads to object storage.

### 2d. CPU and Memory (rough guidelines)

| Throughput class | Cores per broker | Heap (JVM) | OS page cache |
|---|---|---|---|
| < 100 MB/s | 8–12 vCPU | 6 GB | ≥ 32 GB RAM |
| 100–500 MB/s | 16–24 vCPU | 6 GB | ≥ 64 GB RAM |
| > 500 MB/s | 32+ vCPU | 6 GB | ≥ 128 GB RAM |

### 2e. ZooKeeper / KRaft

- **ZooKeeper mode:** 3 or 5 dedicated ZooKeeper nodes (4 vCPU, 8 GB RAM each).
- **KRaft mode (Confluent Platform 7.4+):** No ZooKeeper needed; 3 controller nodes (can be co-located for small clusters, dedicated for large ones).

Present the on-prem sizing as a concise table:

| Parameter | Value |
|---|---|
| Total topics | … |
| Partitions per topic | … |
| Total partitions | … |
| Broker count | … |
| Storage per broker | … |
| Recommended instance type (if AWS/GCP/Azure) | … |

---

## Step 3 — Confluent Cloud Cost Estimation

### 3a. Choose Cluster Type

| Type | Best for | Limits |
|---|---|---|
| **Basic** | Dev / test, low throughput | 250 MB/s ingress, no SLA, single-AZ |
| **Standard** | Production, moderate scale | 250 MB/s ingress, 99.95% SLA, multi-AZ |
| **Dedicated** | High throughput, custom networking, BYOK | Scales to 1 GB/s+ per CKU, 99.99% SLA |
| **Enterprise** (BYOC) | Regulated industries, data residency | Customer-managed VPC |

Select the appropriate tier based on throughput, SLA, and multi-region requirements gathered in Step 1.

### 3b. Confluent Cloud Pricing Components (use current public list prices — note these change; always point the user to https://www.confluent.io/confluent-cloud/pricing/ for the latest)

For **Standard / Basic clusters**, the main cost drivers are:
- **Throughput (CKUs):** Charged per Confluent Kafka Unit-hour. 1 CKU ≈ 50 MB/s ingress + 150 MB/s egress.
- **Storage:** Per GB-month (after free tier).
- **Partitions:** Included up to a limit; overage charged per partition-month.

For **Dedicated clusters:**
- Charged per **CKU-hour** (1 CKU ≈ 50 MB/s ingress). Minimum 1 CKU.
- Storage: Per GB-month (local + tiered object storage).

### 3c. Compute Monthly Estimate

```
cku_needed = ceil(peak_produce_rate_MB_s / 50)   # 50 MB/s per CKU (ingress)
# Also check egress: ceil(peak_consume_rate_MB_s / 150)
# Take the max of ingress and egress CKU counts

monthly_cku_cost    = cku_needed × hours_per_month × cku_hourly_rate
monthly_storage_GB  = produce_rate_MB_s × retention_seconds / 1024
monthly_storage_cost = max(0, monthly_storage_GB - free_storage_GB) × storage_rate_per_GB

total_monthly_estimate = monthly_cku_cost + monthly_storage_cost
```

- `hours_per_month` = 730
- Present a low / mid / high range: low = average throughput, high = sustained peak.

### 3d. Cost Summary Table

Present results as:

| Component | Quantity | Unit price | Monthly cost |
|---|---|---|---|
| CKUs | … | $X/CKU-hr | $… |
| Storage | … GB | $X/GB-mo | $… |
| **Total** | | | **$…** |

Add a note: *Estimates use public list prices. Actual costs vary by region, committed-use discounts, and support tier. Use the [Confluent Cloud pricing calculator](https://www.confluent.io/confluent-cloud/pricing/) for a precise quote.*

---

## Step 4 — Recommendations and Next Steps

After presenting the numbers, provide:

1. **Tier recommendation:** Which deployment model best fits the stated requirements and why.
2. **Key risks:** e.g., under-partitioning causing hot topics, over-retention blowing storage budget, lack of multi-AZ for HA.
3. **Right-sizing tips:** e.g., start with Standard and upgrade to Dedicated if throughput grows beyond 200 MB/s sustained; enable tiered storage to reduce local disk cost.
4. **Links:**
   - Confluent Platform sizing documentation: https://docs.confluent.io/platform/current/kafka/deployment.html
   - Confluent Cloud pricing: https://www.confluent.io/confluent-cloud/pricing/
   - Confluent sizing tool (if available): https://eventsizer.io

---

## Notes

- All formulas are guidance-level estimates. Production clusters require load testing and capacity planning with actual workloads.
- For regulated or financial-services workloads, factor in encryption-at-rest, audit logging, and RBAC overhead (typically +10–15% on broker CPU).
- If the user only needs one of on-prem or cloud sizing, skip the irrelevant section.
