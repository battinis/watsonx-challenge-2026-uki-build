---
name: confluent-sizing
description: Use when the user wants to size a Confluent Kafka cluster or estimate Confluent Cloud costs — covers on-prem broker/partition/storage sizing, Confluent Cloud tier selection, monthly cost estimation, and IBM Confluent part number / pricing guidance for quoting on IBM paper.
---

# Confluent Sizing & IBM Parts Guide

Follow these steps to produce a complete Confluent sizing assessment covering both on-prem cluster sizing and Confluent Cloud cost estimation, and — when the opportunity is on IBM paper — map the result to the correct IBM part numbers and pricing rules.

Source for IBM parts: IBM Confluent Parts and Pricing Guide, 29 May 2026 (IBM & IBM Business Partner Use Only).

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
- Is this for IBM paper (i.e., does the client need an IBM quote/part numbers)?
- Is the client a US Government agency requiring US-Only Support or FedRAMP?
- Is the client on IBM Z / LinuxONE?

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

## Step 4 — IBM Part Number Selection (IBM Paper Only)

Only proceed with this step if the client is purchasing on IBM paper. Use the sizing outputs from Steps 2–3 to recommend the right IBM product and part numbers.

### 4a. Select the Right IBM Offering

Use this decision tree:

| Client scenario | IBM Offering | PID |
|---|---|---|
| Fully managed SaaS, standard | IBM Confluent Cloud | 5900C3E |
| Fully managed SaaS, US Government / FedRAMP | IBM Confluent Cloud for Government | 5900-C3E |
| Client's own cloud (BYOC), Kafka on object storage | IBM WarpStream | 5900-C7D |
| Self-managed, on-prem/off-prem, standard | IBM Confluent Platform | 5900C3D |
| Self-managed, on-prem/off-prem, US Federal (US-Only Support) | IBM Confluent Platform – US-Only Support | 5900C68 |
| Self-managed, IBM Z / LinuxONE | IBM Confluent Platform for Z and LinuxONE | 5900-C3M |
| Self-managed, IBM Z / LinuxONE, US Federal | IBM Confluent Platform for Z and LinuxONE – US-Only Support | 5900-C67 |
| Cloud-native private infrastructure (multi-tenant automation) | IBM Confluent Private Cloud | 5900-C7B |
| Cloud-native private infrastructure, US Government | IBM Confluent Private Cloud – US-Only Support | 5900-C7C |

> **IBM naming vs. Confluent naming:**
> Good (Essentials) = Confluent Basic | Better (Standard) = Confluent Gold | Best (Standard + Advanced Support) = Confluent Platinum

### 4b. IBM Confluent Cloud (SaaS) Part Numbers — PID 5900C3E

**Minimum order:** qty 20 = US$20,000 | **Metric:** Committed Spend (1 unit = US$1,000)
**Support:** Charged against Committed Spend (no separate part); use Business or Premier level only in the Estimator.
**No quotes for China.** Billing: upfront or annual; no auto-renew.

| IBM Part # | Description | Term |
|---|---|---|
| D1546ZX | IBM Confluent Cloud – 12 Mo Term Committed Spend per Annum | 12 mo |
| D1545ZX | IBM Confluent Cloud – 12 Mo Term Committed Spend Overage | 12 mo |
| D154AZX | IBM Confluent Cloud – 24 Mo Term Committed Spend per Annum | 24 mo |
| D1549ZX | IBM Confluent Cloud – 24 Mo Term Committed Spend Overage | 24 mo |
| D154CZX | IBM Confluent Cloud – 36 Mo Term Committed Spend per Annum | 36 mo |
| D154BZX | IBM Confluent Cloud – 36 Mo Term Committed Spend Overage | 36 mo |
| D1547ZX | IBM Confluent Cloud SVC Level Agreement | SLA |

To size the Committed Spend amount, work with your IBM sales contact to use the **Confluent Estimator tool**.

### 4c. IBM Confluent Cloud for Government (FedRAMP) — PID 5900-C3E — USA ONLY

Same metric, same minimum order (qty 20). No quotes for China.

| IBM Part # | Description | Term |
|---|---|---|
| D161TZX | IBM Confluent Cloud for Gov't – 12 Mo Term Committed Spend per Annum | 12 mo |
| D161UZX | IBM Confluent Cloud for Gov't – 12 Mo Term Committed Spend Overage | 12 mo |
| D161VZX | IBM Confluent Cloud for Gov't – 24 Mo Term Committed Spend per Annum | 24 mo |
| D161WZX | IBM Confluent Cloud for Gov't – 24 Mo Term Committed Spend Overage | 24 mo |
| D161XZX | IBM Confluent Cloud for Gov't – 36 Mo Term Committed Spend per Annum | 36 mo |
| D161YZX | IBM Confluent Cloud for Gov't – 36 Mo Term Committed Spend Overage | 36 mo |
| D161SZX | IBM Confluent Cloud for Gov't SVC Level Agreement | SLA |

### 4d. IBM WarpStream (BYOC) — PID 5900-C7D

**Minimum order:** qty 10 = US$10,000 | **Metric:** Committed Spend (1 unit = US$1,000)
For sizing, reach out to the **WarpStream Global Team** with your IBM rep.

| IBM Part # | Description |
|---|---|
| D16J6ZX | IBM WarpStream – 12 Mo Term Committed Spend per Annum |
| D16J7ZX | IBM WarpStream – 12 Mo Term Committed Spend Overage |
| D16J9ZX | IBM WarpStream – 24 Mo Term Committed Spend per Annum |
| D16JAZX | IBM WarpStream – 24 Mo Term Committed Spend Overage |
| D16JBZX | IBM WarpStream – 36 Mo Term Committed Spend per Annum |
| D16JCZX | IBM WarpStream – 36 Mo Term Committed Spend Overage |
| D16J8ZX | IBM WarpStream SVC Level Agreement |

### 4e. IBM Confluent Platform (Self-Managed) — PID 5900C3D

**Minimum (ARR Net after discount):** US$40k (Essentials) / US$50k (Standard)
**Metrics:** Node (software instance on a physical or virtual machine), Connector (standard: packs of 5; premium: individual), Install
**PREPROD / DR (active/passive):** priced at 50% of PROD. Active/active DR → buy PROD part.
**China:** Allowed via Alibaba and Direct.
For sizing, use the **Confluent Estimator** with your IBM sales contact.
Find parts in IBMQ under PID **5900C3D**.

Use the broker count from Step 2 to determine the number of **Node** entitlements needed. Each physical or virtual Kafka broker = 1 Node. Add nodes for ksqlDB, KStreams, Flink, and connector workers as applicable.

**Tier selection:**
- **Essentials** (IBM "Good" / Confluent Basic): no Advanced Support available; min US$40k ARR
- **Standard** (IBM "Better" / Confluent Gold): Advanced Support available; min US$50k ARR

**Core Node parts:**

| IBM Part # | Description |
|---|---|
| D151VZX | Essentials Pre-Production per Node |
| D151TZX | Essentials Production per Node |
| D151UZX | Essentials Disaster Recovery per Node |
| D151XZX | Standard Pre-Production per Node |
| Z151XZX | Standard Pre-Production Advanced Support per Node |
| D151YZX | Standard Production per Node |
| Z151YZX | Standard Production Advanced Support per Node |
| D151WZX | Standard Disaster Recovery per Node |
| Z151WZX | Standard Disaster Recovery Advanced Support per Node |

**Add-on parts (Standard — select as needed based on sizing):**

| Add-on | Pre-Prod | Production | DR |
|---|---|---|---|
| ksqlDB (Essentials) | D1520ZX | D1521ZX | D151ZZX |
| ksqlDB (Standard) | D1523ZX / Z1523ZX | D1524ZX / Z1524ZX | D1522ZX / Z1522ZX |
| KStreams (Essentials) | D1526ZX | D1527ZX | D1525ZX |
| KStreams (Standard) | D1529ZX / Z1529ZX | D152AZX / Z152AZX | D1528ZX / Z1528ZX |
| Connector Pack ×5 (Essentials) | D152CZX | D152BZX | D152DZX |
| Connector Pack ×5 (Standard) | D152FZX / Z152FZX | D152EZX / Z152EZX | D152GZX / Z152GZX |
| Flink Node (Standard only) | D153CZX / Z153CZX | D153EZX / Z153EZX | D153DZX / Z153DZX |
| Flink FS Connector (Standard only) | D153FZX / Z153FZX | D153HZX / Z153HZX | D153GZX / Z153GZX |
| CSFLE (Standard only) | D153TZX / Z153TZX | D153SZX / Z153SZX | D153UZX / Z153UZX |

Premium connectors (Splunk S2S, IBM MQ Sink z/OS, IBM MQ Source z/OS) are sold in units of 1 — refer to the full parts list in the confluent-ppg skill for individual part numbers.

### 4f. IBM Confluent Private Cloud (Self-Managed) — PID 5900-C7B

**Minimum (ARR Net after discount):** US$500k — MUST include Advanced Support.
**China:** Allowed via Alibaba and Direct.
Find parts in IBMQ under **UT30 IBM Confluent Platform (CP)**; use the CP Configurator.

| IBM Part # | Description |
|---|---|
| D16I5ZX / Z16I5ZX | Production per Node / Advanced Support |
| D16I6ZX / Z16I6ZX | Pre-Production per Node / Advanced Support |
| D16I7ZX / Z1617ZX | Disaster Recovery per Node / Advanced Support |

Add connector packs, KStreams, ksqlDB, Flink, and premium connector parts as needed (D16Ixxx / Z16Ixxx series — see confluent-ppg skill for full tables).

### 4g. IBM Confluent Platform for Z and LinuxONE — PID 5900-C3M

**Metric:** VPC (Virtual Processor Core) — not Node.
```
vpc_count = total_vCPUs_allocated_to_Confluent_across_all_ZLinux_LPARs
```
Use ILMT for sub-capacity tracking. Work with Z team PMs for pricing/discounting.
**Perpetual License:** Requires written approval from Jakob Olsen; minimum US$1.2m.
**Important:** Validate each feature is supported on ZLinux before committing — some connectors may be emulation-only.

Core VPC parts: D155TZX (Prod FSL), D155UZX (Prod LIC+12MO), D155VZX (Prod Monthly), D155YZX (PP FSL), D155ZZX (PP LIC+12MO), D1560ZX (PP Monthly) — plus Annual Rnwl E-prefix variants.

---

## Step 5 — IBM Support Tier Guidance

Apply this to all self-managed IBM Confluent offerings.

| ARR Net (after discount) | Support rule |
|---|---|
| < US$250k | No Advanced Support available |
| US$250k – <US$500k | Advanced Support requires written exception from Alex Altman (aaltman@ibm.com); attach to quote |
| ≥ US$500k | Advanced Support available and strongly recommended |
| ≥ US$1m | Advanced Support **should** be included |

**Advanced Support rules:**
- List price = **25% of the product subscription** ('D' part)
- Only available for **Standard** tier ('Z' parts). Not available for Essentials.
- Not available for China sales.
- Quantity and discount on the 'Z' part must **exactly match** the corresponding 'D' part (same qty, same % discount).
- Product subscription must be purchased before or in the same order as Advanced Support.
- Example: D151YZX qty=25 at 30% discount → Z151YZX qty=25 at 30% discount.

**For Cloud offerings (Confluent Cloud / WarpStream):**
- Support is charged against Committed Spend — no separate part number.
- Only use **Business** or **Premier** support levels in the Estimator.

**US-Only Support uplift:** US-Only Support parts carry a **20% uplift** over standard (non-government) parts.

---

## Step 6 — Recommendations and Next Steps

After presenting the numbers, provide:

1. **Tier recommendation:** Which deployment model best fits the stated requirements and why.
2. **Key risks:** e.g., under-partitioning causing hot topics, over-retention blowing storage budget, lack of multi-AZ for HA.
3. **Right-sizing tips:** e.g., start with Standard and upgrade to Dedicated if throughput grows beyond 200 MB/s sustained; enable tiered storage to reduce local disk cost.
4. **IBM quoting reminders (if on IBM paper):**
   - Start IBM quoting early — additional steps are required for approval.
   - Engage the Confluent counterpart early for all opportunities.
   - SaaS provisioning is performed by Confluent for all IBM/IBM BP closed deals.
   - Self-managed software is downloaded via Passport Advantage.
   - No quotes for China for Cloud/WarpStream offerings.
5. **Links:**
   - Confluent Platform sizing docs: https://docs.confluent.io/platform/current/kafka/deployment.html
   - Confluent Cloud pricing: https://www.confluent.io/confluent-cloud/pricing/
   - Confluent sizing tool: https://eventsizer.io
   - IBM Confluent Ecosystem Engagement Guide and Seismic Landing Page (for IBM sellers)

---

## Notes

- All sizing formulas are guidance-level estimates. Production clusters require load testing and capacity planning with actual workloads.
- For regulated or financial-services workloads, factor in encryption-at-rest, audit logging, and RBAC overhead (typically +10–15% on broker CPU).
- If the user only needs on-prem or cloud sizing (not both), skip the irrelevant section.
- IBM part numbers are sourced from the IBM Confluent Parts and Pricing Guide, 29 May 2026. Always verify with the latest IBM Quoting system (IBMQ) before finalising a quote.
- For Expert Labs / professional services: no IBM part numbers exist — an SOW is generated. Contact Hossein Noshin (noshin@ibm.com) or Ted Trask (ttrask@us.ibm.com).
