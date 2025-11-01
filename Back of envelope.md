# Back of Envelope Calculation

## Instagram with 500M DAU

---

## Assumptions

- **DAU**: 500,000,000
- **Writes/user/day**: 1 post
- **Post (write) size**: 1 MB
- **Reads/user/day**: 10 posts
- **Post (read) size**: 100 KB = 0.1 MB
- **Seconds/day**: 86,400

---

## Volumes per Day

### Posts

- **New posts/day**: 500M × 1 = **500M writes/day**
- **Posts read/day**: 500M × 10 = **5B reads/day**

### Data Transfer

**Write data/day:**

```text
500M × 1 MB = 500,000,000 MB = 500 TB/day
```

**Read data/day:**

```text
5B × 0.1 MB = 500,000,000 MB = 500 TB/day
```

### Storage Requirements

**Yearly storage (writes only):**

```text
500 TB/day × 365 = 182,500 TB ≈ 182.5 PB/year
```

> *Note: If you keep all media forever. With 3× replication: ~547.5 PB/year.*

---

## QPS (Queries Per Second)

### Write QPS

```text
500M / 86,400 ≈ 5,787 ops/s (~5.8K/s)
```

### Read QPS

```text
5B / 86,400 ≈ 57,870 ops/s (~58K/s)
```

---

## Throughput (Data per Second)

> **Formula**: Daily → per-second: divide by 86,400

### Write Throughput

```text
500 TB/day ÷ 86,400
= (500,000 GB ÷ 86,400) ≈ 5.79 GB/s
```

### Read Throughput

```text
Same calculation ≈ 5.79 GB/s
```

### Total Data Throughput

```text
≈ 11.6 GB/s (reads + writes)
```

---

## Network Bandwidth (Capacity Needed)

> **Conversion**: GB/s → Gbps by ×8

- **Writes**: 5.79 GB/s × 8 ≈ **46.3 Gbps**
- **Reads**: 5.79 GB/s × 8 ≈ **46.3 Gbps**
- **Aggregate (avg)**: ≈ **92.6 Gbps**

> ⚠️ **Important**: In practice you provision for peak, not average. If peak = 5× average (common for consumer apps), size links and egress paths for **~460 Gbps** aggregate (and distribute across regions/edges).

---

## Quick Summary

| Metric | Value |
|--------|-------|
| **Storage** | 500 TB/day → ~182.5 PB/year (×3 replication → ~547.5 PB/year) |
| **QPS** | ~5.8K writes/s, ~58K reads/s |
| **Throughput** | ~5.8 GB/s writes, ~5.8 GB/s reads (avg) |
| **Bandwidth (avg)** | ~46 Gbps each for reads & writes; ~93 Gbps total |
| **Bandwidth (peak)** | Plan for higher peak (~460 Gbps) |
