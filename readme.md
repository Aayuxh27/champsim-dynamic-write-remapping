# LLC Write Traffic Characterization and Set Remapping for Hotspot Mitigation

A microarchitecture simulation study using ChampSim exploring how write traffic distributes across Last Level Cache sets, and whether remapping strategies can reduce set contention.

---

## Background and Motivation

I started this project while working through a ChampSim assignment from IIT Hyderabad. One of the tasks involved remapping write requests from one LLC set to another, which got me thinking — why does it matter which set a write goes to? And does write traffic actually distribute evenly across sets, or are some sets getting hit much harder than others?

That question turned into this project.

The basic problem: a set-associative LLC has thousands of sets, each holding a fixed number of cache lines (ways). Every memory address maps to exactly one set based on bits from the physical address. If a workload's memory access pattern causes many addresses to map to the same set — intentionally or by chance — that set gets far more write traffic than others. I call these **write hotspots**.

A hotspot set under heavy write pressure has to keep evicting lines to make room for incoming writes. If those evicted lines were reads that the CPU needed again soon, the CPU has to go all the way to DRAM to get them back — roughly 150-180 cycles of wasted time. On top of that, every dirty eviction generates a writeback to DRAM, consuming memory bus bandwidth that could have served useful requests.

The question I wanted to answer: how bad is this imbalance in practice, and can a simple remapping strategy fix it?

---

## Setup

| Parameter | Value |
|---|---|
| Simulator | ChampSim (older version, build-time configuration) |
| LLC size | 4MB (4096 sets, 16 ways, 64B blocks) |
| Replacement policy | LRU |
| Branch predictor | Bimodal |
| Prefetcher | None |
| Warmup | 1M instructions |
| Simulation | 10M instructions |

**Traces used:**

I picked three traces to cover the full range of memory access patterns:

- **462.libquantum** — quantum computing simulation. Very regular, sequential access — reads data once in order and moves on. Light memory pressure.
- **401.bzip2** — data compression. Irregular access over large buffers. Medium memory pressure, working set fits in 4MB.
- **605.mcf_s** — network flow optimization on a large graph. Follows pointer chains through memory randomly. Heavy memory pressure, working set exceeds LLC size.

---

## How the Set Index Works

Before getting into the implementation, it helps to understand how a physical address maps to a cache set.

```
Physical address (64-bit):
[ tag bits | set index (12 bits) | block offset (6 bits) ]
```

For a 4096-set LLC, the set index is the 12 bits above the 6-bit block offset — essentially bits 17:6 of the physical address. Two addresses that share these 12 bits will always map to the same set, regardless of how different they are otherwise.

For random access workloads like mcf, many addresses end up sharing set index bits purely by chance, creating hot sets. For sequential workloads like libquantum, consecutive addresses have consecutive set indices, spreading traffic naturally.

---

## Hotspot Severity Metric

To quantify how uneven the distribution is, I defined:

```
Hotspot Severity = max_writes_in_any_single_set / average_writes_per_set

where average = total_writes / NUM_SET
```

A value of 1.0 = perfectly uniform. Higher values mean more concentrated write pressure.

---

## Phase 0 — Instrumentation

The first step was just measuring the baseline write distribution — no remapping, just counting.

### What I changed

**`inc/cache.h`** — added a per-set write counter as a member of the CACHE class:

```cpp
std::vector<uint64_t> set_write_count;
```

I used `std::vector` instead of a plain array because `NUM_SET` is a runtime value — C++ won't let you size a class member array with a runtime variable. Initialized in the constructor:

```cpp
set_write_count.resize(NUM_SET, 0);
```

**`src/cache.cc`** — inside `handle_writeback()`, right after the set index is computed and before the cache lookup:

```
// pseudocode — handle_writeback() instrumentation

set = get_set(address)

if (cache is LLC) AND (warmup is complete):
    set_write_count[set]++        // count this write

way = check_hit(packet)           // continue normal cache operation
```

The `warmup_complete` gate was important — without it, the write counts include warmup-phase writes and don't match ChampSim's ROI statistics. I noticed this mismatch in an early run and added the gate to fix it.

**`replacement/lru.llc_repl`** — added reporting in `llc_replacement_final_stats()`:

```
// pseudocode — statistics printout

total  = sum of all set_write_count entries
max    = highest count across all sets
min    = lowest count across all sets
avg    = total / NUM_SET
severity = max / avg

print summary

for each set i:
    if set_write_count[i] > 0:
        print set_number and write_count    // skip zero-write sets
```

### Baseline results

| Trace | Total writes | Max (single set) | Hotspot severity | Min |
|---|---|---|---|---|
| bzip2 | 6,616 | 4 | 2.48x | 1 |
| libquantum | 27,948 | 22 | 3.22x | 0 |
| mcf | 26,541 | 60 | 9.26x | 0 |

### What I found

The most striking thing here is that libquantum and mcf have nearly identical total write counts (27,948 vs 26,541) but completely different severity (3.22x vs 9.26x). Same write volume, very different distribution. This told me that severity is driven by access pattern, not how many writes a workload generates.

Looking at the per-set output for mcf, I could see two clusters of hotspot sets — sets 0-38 (low-numbered, mapping to mcf's small metadata structures allocated early in memory) and sets 2500-3600 (mapping to the large graph node array that gets repeatedly traversed). The algorithm visits the same graph nodes over and over, so the same physical addresses — and therefore the same set indices — keep receiving writes.

bzip2 had min = 1, meaning every set got at least one write — a sign of reasonably even distribution. libquantum and mcf both had min = 0, with some sets receiving zero writes while others were heavily loaded.

---

## Phase 1 — Identifying Hotspot and Target Sets

Before building any remapping strategy, I needed to know exactly which sets were hot and which had room to absorb redirected traffic.

From the per-set output, sorted by write count descending, the top 10 hotspot sets for mcf were:

| Rank | Set | Writes |
|---|---|---|
| 1 | 2701 | 60 |
| 2 | 3579 | 58 |
| 3 | 2511 | 58 |
| 4 | 25 | 57 |
| 5 | 2538 | 52 |
| 6 | 3316 | 49 |
| 7 | 2706 | 48 |
| 8 | 2707 | 46 |
| 9 | 17 | 44 |
| 10 | 5 | 43 |

For target sets, I looked for sets with ≤5 writes in the baseline — well below the average of 6.48. There were 2,571 such sets, far more than I needed.

---

## Phase 2 — Static 1-to-1 Remapping

My first attempt was simple: redirect all accesses from each hotspot set to a single low-write target set.

### The coherence requirement

This took me a while to think through correctly. I initially thought about remapping only writes, but that breaks things — if a write goes to set 39 but a subsequent read for the same address still looks in set 2701, it finds nothing and goes all the way to DRAM even though the data is sitting right there in set 39.

The fix: remapping must apply to **all access types** consistently. Since every access — read, write, prefetch — goes through `get_set()` to compute its target set, modifying that single function ensures any address always lands in the same set regardless of whether it's a read or write.

### What I changed

**`inc/cache.h`** — added remapping table:

```cpp
std::vector<uint32_t> remap_table;
```

Initialized with identity mapping (every set maps to itself):

```cpp
remap_table.resize(NUM_SET);
for (uint32_t i = 0; i < NUM_SET; i++) {
    remap_table[i] = i;
}
```

**`src/cache.cc`** — modified `get_set()`:

```
// pseudocode — get_set() with static remapping

set = lower 12 bits of block address    // original computation

if cache is LLC:
    set = remap_table[set]              // apply remapping (identity by default)

return set
```

**`replacement/lru.llc_repl`** — populated the table in `llc_initialize_replacement()`:

```
remap_table[2701] = 39
remap_table[3579] = 40
remap_table[2511] = 41
remap_table[25]   = 42
remap_table[2538] = 43
remap_table[3316] = 49
remap_table[2706] = 51
remap_table[2707] = 59
remap_table[17]   = 60
remap_table[5]    = 64
```

### Phase 2 results vs baseline

| Metric | Baseline | Phase 2 | Change |
|---|---|---|---|
| IPC | 0.393141 | 0.393018 | -0.03% |
| Max writes | 60 | 60 | none |
| Hotspot severity | 9.26x | 9.26x | none |

### What I found — and why it failed

The writes moved exactly where I told them to go. Set 2701 disappeared from the output and set 39 showed up with 60 writes. The mechanism was correct.

But severity didn't change at all. I just moved the hotspot from set 2701 to set 39. Set 39 now has exactly as much write pressure as set 2701 had before.

The reason is obvious in retrospect — moving all 60 writes from one set to another creates a new hotspot at the target with identical pressure. To actually reduce severity you have to **split** the writes across multiple target sets so no single set absorbs all of them. Phase 2 proved the mechanism works but also proved that 1-to-1 remapping is hotspot relocation, not hotspot mitigation.

---

## Phase 3 — Split Remapping Across 8 Target Sets

The idea: instead of mapping hotspot set 2701 entirely to set 39, distribute its traffic across 8 target sets based on additional address bits.

### How the splitting works

The set index uses bits 17:6 of the physical address. The next 3 bits above that (bits 20:18) are currently unused for set indexing — they go into the tag. I use these 3 bits as a "slot" index to pick among 8 target sets:

```
slot = bits 20:18 of block address    // values 0-7
final_set = split_targets[original_set][slot]
```

Because the slot comes from fixed address bits, any two accesses to the same physical address always get the same slot — coherence is preserved automatically.

### Expected writes per target set

For the hottest set (2701, 60 writes):
```
60 writes / 8 targets = 7.5 writes per target
```
Near the average of 6.48. No new hotspots created.

### Target set selection

I needed 80 unique target sets (10 hotspots × 8 each) with ≤5 baseline writes. There were 2,571 candidates — plenty of room. I assigned them sequentially:

| Hotspot | Writes | Target sets |
|---|---|---|
| 2701 | 60 | 39, 40, 41, 42, 43, 46, 49, 51 |
| 3579 | 58 | 55, 58, 59, 60, 64, 65, 66, 67 |
| 2511 | 58 | 68, 69, 70, 71, 72, 73, 74, 76 |
| 25 | 57 | 77, 78, 79, 80, 81, 82, 83, 84 |
| 2538 | 52 | 85, 86, 87, 88, 89, 90, 91, 92 |
| 3316 | 49 | 93, 94, 95, 96, 97, 99, 100, 101 |
| 2706 | 48 | 102, 103, 104, 105, 106, 107, 108, 109 |
| 2707 | 46 | 110, 111, 112, 113, 116, 117, 118, 119 |
| 17 | 44 | 123, 127, 128, 129, 130, 131, 132, 133 |
| 5 | 43 | 134, 135, 136, 137, 138, 139, 140, 141 |

### What I changed

**`inc/cache.h`** — added split targets structure:

```cpp
std::vector<std::vector<uint32_t>> split_targets;
```

Initialized as empty vectors (non-hotspot sets get no entry):

```cpp
split_targets.resize(NUM_SET);
```

**`src/cache.cc`** — updated `get_set()`:

```
// pseudocode — get_set() with split remapping

set = lower 12 bits of block address

if cache is LLC:
    if split_targets[set] is not empty:
        slot = bits 20:18 of block address    // 3 bits = 8 possible targets
        return split_targets[set][slot]
    return remap_table[set]                   // identity for non-hotspots

return set
```

**`replacement/lru.llc_repl`** — populated split_targets in `llc_initialize_replacement()`:

```
split_targets[2701] = {39,40,41,42,43,46,49,51}
split_targets[3579] = {55,58,59,60,64,65,66,67}
// ... and so on for all 10 hotspot sets
```

### Phase 3 results

| Metric | Baseline | Phase 2 | Phase 3 |
|---|---|---|---|
| IPC | 0.393141 | 0.393018 | 0.393052 |
| Max writes | 60 | 60 | **43** |
| Hotspot severity | 9.26x | 9.26x | **6.64x** |
| LLC hits | 178,031 | 177,961 | 177,991 |
| LLC misses | 114,327 | 114,397 | 114,367 |
| DRAM WQ writes | 557 | 562 | 555 |

### What I found

Split remapping successfully reduced severity from 9.26x to 6.64x — a 28% improvement. Max writes dropped from 60 to 43. The mechanism works correctly.

But IPC didn't improve meaningfully.

This took some investigation to understand. Looking at the LLC writeback statistics:

```
LLC WRITEBACK ACCESS: 26,541   HIT: 26,498   MISS: 43
```

Only 43 out of 26,541 writes caused actual evictions. The other 99.8% hit lines already present in LLC and just updated the dirty bit — no eviction, no performance impact. The write hotspot at set 2701 had 60 writes, but most of those were hitting cached lines, not throwing out useful read data.

The real bottleneck was always the read side:

```
LLC LOAD ACCESS: 248,963   HIT: 134,789   MISS: 114,174
```

114,174 load misses going to DRAM — all from mcf's pointer chasing, where each address is only known after the previous one resolves. Even perfect write distribution wouldn't touch these. Write hotspot mitigation helps when writes are actually evicting useful reads. For mcf at 4MB, they mostly weren't.

---

## Phase 4 — Cross-Trace Validation

I ran bzip2 and libquantum through the same mcf-tuned Phase 3 build to see what happens when you apply a workload-specific remapping to a different workload.

### Results

| Trace | Baseline severity | Phase 3 severity | Change |
|---|---|---|---|
| mcf | 9.26x | 6.64x | -28% |
| libquantum | 3.22x | 3.22x | 0% |
| bzip2 | 2.48x | 2.48x | 0% |

The mcf-tuned table had zero effect on the other two workloads. The remapped sets (5, 17, 25, 2511...) don't accumulate significant write traffic in bzip2 or libquantum, so redirecting them changed nothing.

This result directly shows the limitation of static workload-specific remapping — it cannot generalize. A table built from one workload's profiling data is useless, or potentially harmful, for a different workload. The solution would be dynamic remapping that observes runtime write patterns and updates targets accordingly — which is the planned next phase.

---

## Summary of Findings

**Finding 1 — Hotspot severity is driven by access pattern, not write volume**

libquantum and mcf have nearly identical write volumes (27,948 vs 26,541) but severity of 3.22x vs 9.26x. Sequential access distributes writes evenly. Random pointer-chasing creates hotspots by chance concentration of set index bits.

**Finding 2 — 1-to-1 remapping relocates hotspots, it does not mitigate them**

Phase 2 moved all 60 writes from set 2701 to set 39, creating a new hotspot at set 39 with identical pressure. Severity stayed at 9.26x. The lesson: you need to split writes across multiple targets to actually reduce the maximum.

**Finding 3 — Split remapping reduces severity but not necessarily IPC**

Phase 3 reduced severity 28% (9.26x → 6.64x). But IPC was flat because mcf's writes overwhelmingly hit in LLC rather than causing evictions. The real bottleneck for mcf is 114K serial read misses from pointer chasing — write remapping doesn't help that.

**Finding 4 — Static remapping does not generalize across workloads**

A table built from mcf profiling had zero effect on bzip2 and libquantum. This motivates dynamic epoch-based remapping as a natural next step.

---

## Limitations

- The remapping table was profiled from mcf — it is workload-specific and cannot generalize
- ChampSim models fixed LLC latency (20 cycles) regardless of remapping overhead — real hardware would have a small table lookup cost
- Only the top 10 hotspot sets were remapped — sets 11-N remained unremediated (set 2512 became the new maximum at 43 writes)
- Single-core simulation — multi-core coherence effects not studied
- NVM wear is not directly simulated — write count distribution is used as a proxy

---

## What's Next

**Dynamic epoch-based remapping** — instead of a fixed offline-profiled table, maintain runtime write counters per set and update the remapping table every N cycles based on what's actually happening. This is hardware-realistic and workload-agnostic. The `set_write_count[]` infrastructure is already in place — the remaining piece is the epoch trigger and runtime table update logic.

**Workload with higher dirty eviction rate** — mcf's near-zero dirty eviction rate meant write remapping had no performance impact. Running the same study on a workload where writes are actually evicting useful reads would show the conditions under which hotspot mitigation matters for IPC.

**Extending to gem5 with NVM modeling** — write count distribution as a proxy for NVM wear is interesting but indirect. gem5 with a PCM or ReRAM model would let you directly simulate cell endurance and show concrete wear lifetime improvements from balanced write distribution.

---

## Build and Run

```bash
# set LLC to 4MB in inc/cache.h
# LLC_SET NUM_CPUS*4096
# LLC_WAY 16

./build_champsim.sh bimodal no no no no lru 1

./run_champsim.sh bimodal-no-no-no-no-lru-1core 1 10 605.mcf_s-665B.champsimtrace.xz
./run_champsim.sh bimodal-no-no-no-no-lru-1core 1 10 401.bzip2-277B.champsimtrace.xz
./run_champsim.sh bimodal-no-no-no-no-lru-1core 1 10 462.libquantum-1343B.champsimtrace.xz
```

Write distribution summary and per-set counts print automatically at simulation end.

---

## Repository Structure

```
llc-write-hotspot-mitigation/
├── README.md
├── inc/
│   └── cache.h              # set_write_count, remap_table, split_targets
├── src/
│   └── cache.cc             # get_set() modified, handle_writeback() instrumented
├── replacement/
│   └── lru.llc_repl         # llc_initialize_replacement(), llc_replacement_final_stats()
└── results/
    ├── phase0_baseline/
    │   ├── mcf.txt
    │   ├── bzip2.txt
    │   └── libquantum.txt
    ├── phase2_static_remap/
    │   └── mcf.txt
    └── phase3_split_remap/
        ├── mcf.txt
        ├── bzip2.txt
        └── libquantum.txt
```