# LLC Write Traffic Characterization and Set Remapping for Hotspot Mitigation

**Simulator**: ChampSim (cycle-accurate microarchitecture simulator)  
**Language**: C++  
**LLC Configuration**: 4MB, 16-way, 64B block size, LRU replacement  
**Workload**: 1M warmup + 10M simulation instructions, no prefetcher, bimodal branch predictor

---

## Project Overview

Modern set-associative caches use a simple modulo-based indexing function to map memory addresses to cache sets. A side effect of this scheme is that certain workloads generate disproportionately high write traffic to a small subset of cache sets — called **write hotspots** — while the majority of sets sit underutilized.

This project:
1. Instruments the LLC write pipeline to measure per-set write traffic
2. Quantifies hotspot severity across different workload classes
3. Evaluates remapping strategies that redistribute write traffic to reduce hotspot pressure

---

## The Problem

A 4MB LLC with 4096 sets and 16 ways gives each set exactly 16 ways to hold data. Every write request maps to a specific set determined by bits 17:6 of the physical address:

```
Physical address:
[ 63 ........... 18 | 17 .... 6 | 5 ... 0 ]
       tag            set index    block offset
```

When many addresses share the same set index bits, that set accumulates far more writes than others. When a set overflows — all 16 ways occupied — the cache must evict a line. If that line is dirty, it generates a writeback to DRAM, consuming memory bus bandwidth. If the evicted line was a frequently-read line, subsequent reads miss and go to DRAM at 150-180 cycle latency.

**The core insight**: this is not a capacity problem — it is a distribution problem. Redistributing writes more evenly across sets reduces premature evictions and DRAM writeback traffic.

---

## Traces Used

| Trace | Type | Memory behavior |
|---|---|---|
| 462.libquantum-1343B | Light | Sequential streaming, data accessed once and never reused |
| 401.bzip2-277B | Medium | Irregular access over large buffers, working set fits in 4MB |
| 605.mcf_s-665B | Heavy | Random pointer-chasing through large graph, working set exceeds LLC |

---

## Hotspot Severity Metric

```
Hotspot Severity = max_writes_in_any_single_set / average_writes_per_set

where:
    average_writes_per_set = total_writes / NUM_SET
```

A value of 1.0 means perfectly uniform distribution.
A value of 9.0 means the hottest set received 9x more writes than average.

---

## Phase 0 — Baseline Instrumentation

### Goal
Add per-set write counters to the LLC. Measure the write distribution across all sets for each workload with no remapping. Pure observation.

### Files Changed

**`inc/cache.h`** — added two member variables to the CACHE class:
```cpp
std::vector<uint64_t> set_write_count;  // write counter per set
```
Initialized in the CACHE constructor:
```cpp
set_write_count.resize(NUM_SET, 0);
```
`std::vector` is used instead of a plain array because `NUM_SET` is a runtime value — C++ does not allow plain arrays sized by runtime variables inside a class.

---

**`src/cache.cc`** — inside `handle_writeback()`, after `get_set()` computes the set index and before `check_hit()`:

```
PSEUDOCODE — handle_writeback() instrumentation:

set = get_set(address)           // compute set index from address

if cache is LLC AND warmup is complete:
    set_write_count[set]++       // count this write

way = check_hit(packet)          // proceed with normal cache lookup
```

The `warmup_complete` gate ensures only simulation-phase writes are counted, matching ChampSim's ROI statistics exactly. Without this gate, warmup writes inflate the total and don't match the LLC WRITEBACK ACCESS count in the output.

---

**`replacement/lru.llc_repl`** — inside `llc_replacement_final_stats()`:

```
PSEUDOCODE — write distribution reporting:

total_writes = sum of all set_write_count entries
max_writes   = highest single set count
min_writes   = lowest single set count
avg_writes   = total_writes / NUM_SET
severity     = max_writes / avg_writes

print summary (total, max, min, avg, severity)

for each set i from 0 to NUM_SET-1:
    if set_write_count[i] > 0:
        print set_number and write_count
```

Only non-zero sets are printed to keep output manageable.

### Phase 0 Results

| Trace | Total writes | Max (single set) | Hotspot severity | Min |
|---|---|---|---|---|
| bzip2 | 6,616 | 4 | 2.48x | 1 |
| libquantum | 27,948 | 22 | 3.22x | 0 |
| mcf | 26,541 | 60 | 9.26x | 0 |

### Phase 0 Findings

**Hotspot severity is driven by access pattern, not write volume.**
libquantum and mcf have nearly identical total write counts (27,948 vs 26,541) but completely different severity (3.22x vs 9.26x). libquantum's sequential streaming spreads writes evenly across sets. mcf's random pointer-chasing causes writes to cluster by chance into specific sets.

**bzip2 shows low severity AND low total writes.**
At 4MB, bzip2's working set fits in the LLC. Lines get reused before eviction, so dirty writebacks are rare — only 6,616 total. With few writes spread across 4096 sets, no single set accumulates much pressure. min = 1 means every set got at least one write, a signature of even distribution.

**mcf has two distinct hotspot clusters:**
Running `sort -k2 -rn` on per-set output reveals hotspots concentrated in:
- Sets 0–38 (low-numbered) — maps to mcf's small metadata/header structures allocated early in virtual address space at low physical addresses
- Sets 2500–3600 (mid-range) — maps to mcf's large graph node array, repeatedly traversed by the pointer-chasing algorithm

The same graph nodes get visited on every algorithm iteration, so the same physical addresses — and therefore the same set indices — keep receiving writes.

---

## Phase 1 — Identifying Hotspot and Target Sets

### Goal
Extract the specific set numbers that are hot (candidates for remapping away from) and cold (candidates to receive redirected traffic).

### Method
From the per-set output, sort by write count descending to find hotspot sets. Find gaps in the output (sets with zero writes) for target sets.

```
PSEUDOCODE — identify hotspot sets:

sort per_set_write_counts by count descending
top_N_hotspots = first N entries

PSEUDOCODE — identify target sets:

for i from 0 to NUM_SET-1:
    if i not in per_set_write_counts:
        add i to target_set_list
```

### Phase 1 Results for mcf

**Top 10 hotspot sets:**

| Rank | Set | Writes | Ratio to average (6.48) |
|---|---|---|---|
| 1 | 2701 | 60 | 9.26x |
| 2 | 3579 | 58 | 8.95x |
| 3 | 2511 | 58 | 8.95x |
| 4 | 25 | 57 | 8.80x |
| 5 | 2538 | 52 | 8.02x |
| 6 | 3316 | 49 | 7.56x |
| 7 | 2706 | 48 | 7.41x |
| 8 | 2707 | 46 | 7.10x |
| 9 | 17 | 44 | 6.79x |
| 10 | 5 | 43 | 6.64x |

**First 10 zero-write target sets:** 39, 40, 41, 42, 43, 49, 51, 59, 60, 64

---

## Phase 2 — Static 1-to-1 Remapping

### Goal
Redirect all accesses destined for hotspot sets to zero-write target sets. Measure the effect on write distribution and performance.

### Design Decision — Modifying `get_set()`

Remapping must apply to **all access types** — reads, writes, and prefetches — consistently. If only writes are remapped but reads still go to the original set, a read for data written to the target set will miss and go to DRAM unnecessarily.

Since every access type calls `get_set()` to compute its target set, modifying this single function ensures consistent remapping for all traffic.

### Files Changed

**`inc/cache.h`** — added remapping table member variable:
```cpp
std::vector<uint32_t> remap_table;  // Phase 2: static remapping table
```
Initialized in the CACHE constructor with identity mapping (every set maps to itself by default):
```cpp
remap_table.resize(NUM_SET);
for (uint32_t i = 0; i < NUM_SET; i++) {
    remap_table[i] = i;
}
```

---

**`src/cache.cc`** — modified `get_set()`:

```
PSEUDOCODE — get_set() with static remapping:

compute set = lower 12 bits of block address    // original indexing

if cache is LLC:
    set = remap_table[set]                      // apply remapping if entry differs

return set
```

Since `remap_table[i] = i` by default, non-remapped sets pass through unchanged.

---

**`replacement/lru.llc_repl`** — populated remapping table in `llc_initialize_replacement()`:

```
PSEUDOCODE — static remapping table setup:

remap_table[2701] = 39    // hotspot set → zero-write target set
remap_table[3579] = 40
remap_table[2511] = 41
remap_table[25]   = 42
remap_table[2538] = 43
remap_table[3316] = 49
remap_table[2706] = 51
remap_table[2707] = 59
remap_table[17]   = 60
remap_table[5]    = 64

all other entries remain identity (remap_table[i] = i)
```

### Phase 2 Results vs Baseline

| Metric | Baseline | Phase 2 | Change |
|---|---|---|---|
| IPC | 0.393141 | 0.393018 | -0.03% |
| LLC hits | 178,031 | 177,961 | -70 |
| LLC misses | 114,327 | 114,397 | +70 |
| Max writes (single set) | 60 | 60 | unchanged |
| Hotspot severity | 9.26x | 9.26x | unchanged |

### Phase 2 Findings — Critical Result

**1-to-1 static remapping is hotspot relocation, not hotspot mitigation.**

The writes moved exactly as intended — confirmed by checking that hotspot sets (2701, 3579, 2511, 25, 2538...) disappeared from the output and target sets (39, 40, 41, 42, 43...) now carry exactly those write counts. The mechanism is correct.

But hotspot severity did not change. Set 39 now has 60 writes instead of set 2701. The max write count is still 60. Severity is still 9.26x. The problem was relocated, not solved.

**Why this happens:**
Moving all 60 writes from one set to another creates a new hotspot with identical pressure at the target location. To actually reduce severity, writes must be **split across multiple target sets** so no single target absorbs the full load. If 60 writes are distributed across 4 target sets, each receives ~15 — well below average.

**Performance impact:**
Near-zero IPC change (-0.03%) because severity did not change. The same eviction pressure exists at different sets. This result directly motivates Phase 3.

---

## Phase 3 — Split Remapping (Planned)

### Goal
Instead of mapping each hotspot set entirely to one target, distribute writes from a hotspot set across multiple target sets based on additional address bits. This actually reduces the max write count per set and lowers severity.

### Design

```
PSEUDOCODE — split remapping in get_set():

compute set = lower 12 bits of block address

if cache is LLC AND set is a hotspot set:
    base_target = remap_table[set]           // base target from table
    offset      = (address >> 12) & 3        // use next 2 address bits
    set         = base_target + offset       // distribute across 4 target sets

return set
```

For set 2701 → base target 39:
- Addresses where bits 13:12 = 00 → set 39
- Addresses where bits 13:12 = 01 → set 40
- Addresses where bits 13:12 = 10 → set 41
- Addresses where bits 13:12 = 11 → set 42

60 writes split across 4 sets → ~15 writes per set → below the 6.48 average.

### Expected Outcome
- Max writes per set drops significantly
- Hotspot severity drops from 9.26x toward ~2-3x
- IPC improvement measurable if eviction pressure on original hotspot sets was causing real performance loss

---

## Key Learnings

**Hotspot severity and write volume are independent.** Two traces with similar total write counts can have completely different severity depending on their access patterns. Characterizing a workload requires both metrics.

**1-to-1 remapping proves the mechanism but not the benefit.** Phase 2 is a necessary step — it confirms that remapping works correctly before adding splitting complexity. The null result is itself a finding: you cannot reduce hotspot severity without splitting.

**get_set() is the single right interception point.** Because all access types — reads, writes, prefetches — call this function, a single modification there ensures coherent remapping. Intercepting only write traffic would break read-write coherence.

**Performance impact requires severity reduction.** IPC only improves when eviction pressure actually decreases. Relocating a hotspot achieves nothing for performance. This is why Phase 3 (splitting) is necessary to see measurable IPC gains.

---

## Repository Structure

```
llc-write-hotspot-mitigation/
├── README.md
├── inc/
│   └── cache.h              # set_write_count vector, remap_table vector
├── src/
│   └── cache.cc             # get_set() modified, handle_writeback() instrumented
├── replacement/
│   └── lru.llc_repl         # llc_initialize_replacement(), llc_replacement_final_stats()
└── results/
    ├── phase0_baseline/
    │   ├── mcf.txt
    │   ├── bzip2.txt
    │   └── libquantum.txt
    └── phase2_static_remap/
        └── mcf.txt
```

---

## Build and Run

```bash
# build
./build_champsim.sh bimodal no no no no lru 1

# run
./run_champsim.sh bimodal-no-no-no-no-lru-1core 1 10 605.mcf_s-665B.champsimtrace.xz
./run_champsim.sh bimodal-no-no-no-no-lru-1core 1 10 401.bzip2-277B.champsimtrace.xz
./run_champsim.sh bimodal-no-no-no-no-lru-1core 1 10 462.libquantum-1343B.champsimtrace.xz
```

The write distribution summary and per-set counts print automatically at the end of each run.

---

## Known Limitations

- Write counters count LLC-level writebacks only — RFO traffic at L1D that misses and reaches LLC as a read is not counted as a write at the LLC level
- ChampSim models fixed LLC access latency (20 cycles) regardless of set remapping — real hardware would incur a small table lookup overhead
- Static remapping uses offline profiling data from mcf — the same table may not generalize to other traces or other phases of the same trace
- NVM wear is not directly simulated — write count distribution serves as a proxy for wear unevenness