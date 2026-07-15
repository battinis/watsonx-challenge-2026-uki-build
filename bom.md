# Confluent Sizing — July 15, 2026

> Sizing is guidance-level. Production clusters require load testing with actual workloads.
> IBM part numbers sourced from IBM Confluent Parts and Pricing Guide, 29 May 2026. Verify in IBMQ before finalising a quote.

---

## Workload Inputs

| Parameter | Value |
|---|---|
| Peak produce rate | 50 MB/s |
| Average message size | 1 KB |
| Number of topics | 20 |
| Replication factor | 3 |
| Consumer fan-out ratio | 2× (2 consumer groups) |
| Retention period | 7 days |
| Tiered storage | No |
| Target availability | 99.9% |
| Multi-AZ | Yes |
| Deployment model | Confluent Cloud |
| Cloud provider & region | AWS — us-east-1 |
| IBM paper | Yes |
| US Government / FedRAMP | No |
| IBM Z / LinuxONE | No |

---

## On-Prem / Confluent Platform Sizing

> Included as a reference baseline. Primary deployment is Confluent Cloud (see next section).

### Sizing Workings

| Step | Formula | Result |
|---|---|---|
| Partitions per topic | ceil(50 MB/s ÷ 10 MB/s per partition) | **5** |
| Total partitions | 5 × 20 topics | **100** |
| Min brokers (partition cap) | ceil(100 ÷ 4,000) = 1 → floor to min 3 | **3** |
| +1 rolling-maintenance spare | 3 + 1 | **4 brokers** |
| Per-broker data rate | (50 MB/s × 3) ÷ 4 | **37.5 MB/s** ✓ (< 80% of 1 GbE) |
| Total raw storage | 50 MB/s × 604,800 s × 3 ÷ 1,024 | **88,594 GB (~86.5 TB)** |
| Storage per broker (raw) | 88,594 ÷ 4 | **22,148 GB** |
| Storage per broker (+20% headroom) | 22,148 × 1.20 | **≈ 26,578 GB (26.0 TB)** |

> ⚠️ **Storage note:** 26 TB per broker for 7-day retention on 50 MB/s with RF3 is significant. Strongly recommend enabling **tiered storage** to offload to object storage (local hot tier: 1–2 days) — this would reduce local disk to ~7–8 TB per broker.

### On-Prem Summary

| Parameter | Value |
|---|---|
| Total topics | 20 |
| Partitions per topic | 5 |
| Total partitions | 100 |
| Broker count | 4 (3 production + 1 spare) |
| Per-broker throughput | 37.5 MB/s |
| Storage per broker | ~26 TB (raw + 20% headroom) |
| CPU per broker | 8–12 vCPU |
| RAM per broker | ≥ 32 GB |
| JVM heap | 6 GB |
| Controller mode | KRaft (Confluent Platform 7.4+) — no ZooKeeper required |
| Recommended instance (AWS) | r6i.2xlarge or i3en.2xlarge (NVMe for high I/O) |

---

## Confluent Cloud Cost Estimate

**Cluster type:** Standard (multi-AZ, 99.95% SLA) — AWS us-east-1
**Pricing basis:** Public list prices as of July 2026. Region: AWS us-east-1.

### CKU Sizing

| Direction | Rate | CKU capacity | CKUs required |
|---|---|---|---|
| Ingress (produce) | 50 MB/s | 50 MB/s per CKU | **1 CKU** |
| Egress (consume) | 100 MB/s (2× fan-out) | 150 MB/s per CKU | **1 CKU** |
| **CKUs needed** | | | **1 CKU** |

### Monthly Cost Breakdown

| Component | Quantity | Unit price | Monthly cost |
|---|---|---|---|
| CKUs | 1 CKU × 730 hrs | $1.50 / CKU-hr | **$1,095** |
| Storage | 29,531 GB | $0.10 / GB-month | **$2,953** |
| **Total (list price)** | | | **$4,048 / month** |

> **Storage:** 50 MB/s × 604,800 s ÷ 1,024 ≈ 29,531 GB retained per month (no tiered storage).
> With **tiered storage** (2-day hot tier): local storage drops to ~8,437 GB → storage cost ~$844/mo → **total ≈ $1,939/mo**.

### Annual Committed Spend

| Scenario | Monthly | Annual | Committed Spend units (qty) |
|---|---|---|---|
| No tiered storage (current) | $4,048 | **$48,578** | **49 units** |
| With tiered storage (recommended) | $1,939 | **$23,268** | **24 units** |

*1 Committed Spend unit = US$1,000. Minimum order: 20 units.*

---

## IBM Part Numbers (IBM Paper — PID 5900C3E)

**Offering:** IBM Confluent Cloud (SaaS) — Standard cluster
**Metric:** Committed Spend (1 unit = US$1,000) | **Minimum order:** qty 20
**Support:** Charged against Committed Spend (no separate part number). Use **Business** or **Premier** support level in the Estimator only.

### Recommended Parts — 12-Month Term

| IBM Part # | Description | Qty (units) | Notes |
|---|---|---|---|
| **D1546ZX** | IBM Confluent Cloud – 12 Mo Term Committed Spend per Annum | **49** (no tiered storage) / **24** (with tiered storage) | Annual committed spend |
| **D1545ZX** | IBM Confluent Cloud – 12 Mo Term Committed Spend Overage | As needed | Overage beyond committed amount |
| **D1547ZX** | IBM Confluent Cloud SVC Level Agreement | 1 | SLA entitlement |

### Optional Longer-Term Parts

| IBM Part # | Description | Term |
|---|---|---|
| D154AZX | IBM Confluent Cloud – 24 Mo Term Committed Spend per Annum | 24 mo |
| D1549ZX | IBM Confluent Cloud – 24 Mo Term Committed Spend Overage | 24 mo |
| D154CZX | IBM Confluent Cloud – 36 Mo Term Committed Spend per Annum | 36 mo |
| D154BZX | IBM Confluent Cloud – 36 Mo Term Committed Spend Overage | 36 mo |

> Use the **Confluent Estimator** tool (with your IBM sales contact) to confirm the exact Committed Spend amount before quoting.

---

## Recommendations & Next Steps

### Tier Recommendation
**Confluent Cloud — Standard cluster (multi-AZ, AWS us-east-1)** is the right fit:
- 50 MB/s produce rate is well within Standard limits (250 MB/s max).
- Multi-AZ satisfies 99.9% availability target (Standard SLA: 99.95%).
- No need for Dedicated cluster unless throughput grows beyond ~200 MB/s sustained or custom networking / BYOK is required.

### Key Risks

| Risk | Detail |
|---|---|
| 🔴 Storage cost dominates | Without tiered storage, 29,531 GB/month at $0.10/GB = $2,953/mo — 73% of total bill. Enable tiered storage immediately. |
| 🟡 Under-partitioned topics | 5 partitions/topic limits per-topic parallelism. Review if any single topic needs higher throughput. |
| 🟢 CKU count is minimal | 1 CKU is sufficient for current load. Burst headroom exists up to 250 MB/s before needing to scale. |

### Right-Sizing Tips

1. **Enable tiered storage** — cuts monthly cost from ~$4,048 to ~$1,939, reducing the Committed Spend from 49 units to 24 units.
2. **Monitor CKU utilisation** — if sustained produce rate exceeds 50 MB/s, add a second CKU.
3. **Review partition count** — if a topic becomes a bottleneck, increase to 10–20 partitions before it impacts latency.
4. **Upgrade to Dedicated** only if: throughput > 200 MB/s sustained, BYOK encryption required, or private networking (VPC peering / PrivateLink) is needed.

### IBM Quoting Reminders

- Start IBM quoting early — additional approval steps are required.
- Engage the **Confluent counterpart** early for all opportunities.
- **SaaS provisioning** is performed by Confluent for all IBM / IBM BP closed deals.
- No quotes for **China** on IBM Confluent Cloud.
- Support is billed against Committed Spend — use **Business** or **Premier** levels in the Estimator only.

### Useful Links

- [Confluent Cloud pricing calculator](https://www.confluent.io/confluent-cloud/pricing/)
- [Confluent Platform sizing docs](https://docs.confluent.io/platform/current/kafka/deployment.html)
- [Confluent sizing tool — eventsizer.io](https://eventsizer.io)
- IBM Confluent Ecosystem Engagement Guide and Seismic Landing Page (IBM sellers only)
