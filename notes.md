# 1 Introduction

Cache timing channels have grown into a critical security concern in modern processors. By exploiting observable latency differences in cache accesses, attackers can extract sensitive information (side-channel attacks) or coordinate hidden communication (covert channels), all without standard OS-mediated interfaces. Although a number of defenses and mitigations have been proposed, the arms race remains ongoing.

This survey offers a detailed, structured review of the state-of-the-art cache timing channel attacks from roughly the past five years (excluding transient-execution exploits like Spectre or Meltdown, and DRAM-based exploits such as RowHammer) and discusses the efficacy of existing countermeasures.

# 2 Background and Terminology

## 2.1 Covert Channels and Side Channels

- **Covert Channels:** In a covert channel, two colluding processes (often called trojan and spy) exfiltrate data through microarchitectural elements. For example, one process can encode bits by modulating cache states, while another process decodes them by measuring the induced timing differences.
- **Side Channels:** A side channel arises when a victim process unwittingly leaks internal state through observable effects (e.g., timing, power, or electromagnetic emissions). In cache-based side channels, an attacker infers the victim's secret information (such as cryptographic keys) solely by measuring how cache hits/misses affect access latency.

## 2.2 Threat Model

Typical cache timing attacks assume that:

1. The attacker can execute unprivileged code on the same physical machine (often on a separate core).
2. The attacker possesses a measurement mechanism for timing events, typically a high-precision timer (e.g., `rdtsc`, performance counters, or equivalent).
3. The attacker can locate or interact with memory that maps to the same cache sets as the victim. Shared memory (like shared libraries or deduplicated pages) generally makes attacks easier.

## 2.3 Cache Attacks Research Directions

In the realm of pure cache timing channel attacks, recent academic research tends to concentrate on the following main directions:

1. **_Exploiting More Cache Timing Channels_**
   While traditional cache attacks focus on basic "hit/miss" differences or leveraging flush instructions, more recent work increasingly targets cache coherence protocols, replacement-policy metadata (e.g., pseudo-LRU or random policies), dirty bits, etc.
2. **_Methodological Diversity for Exploiting Timing Channels_**
   Rather than discovering new channels, researchers develop diverse methods to exploit the same fundamental timing channel under different conditions and requirements.
3. **_Increasing Bandwidth and Reducing Error Rates_**  
   To enhance cache timing channels for data exfiltration or covert communication, researchers focus on higher throughput and lower error rates. Tactics include:
   - Employing statistical methods or machine learning to mitigate jitter in real time.
   - Designing innovative probing or prefetch patterns to distinguish subtle cache states and minimize interference.
4. **_Real-World Applications and Threat Evaluations_**  
   Many recent publications integrate cache timing side-channel techniques into real-world scenarios to highlight their practical impact:
   - Multi-tenant cloud environments to bypass seemingly rigid isolation.
   - JavaScript-based website fingerprinting or user tracking via cache occupancy measurements.
   - Demonstrating real attacks on cryptographic libraries, private messaging applications (e.g., Signal), or cryptocurrency wallets.

# 3 Cache Characteristics

Modern processor cache architectures incorporate multiple levels and complex organizations that attackers can exploit. This section details the key characteristics relevant to cache timing attacks.

## 3.1 Cache Structure and Organization

### 3.1.1 Basic Cache Components

- **Cache Lines:** Fundamental unit of cache storage (typically 64 bytes)
- **Sets:** Collection of cache lines that can store data from different memory addresses
- **Ways:** Number of cache lines in each set (N-way set-associative)
- **Tag Store:** Metadata storing address information for cached data
- **Data Store:** Actual cached data content
- **Metadata:** Metadata including replacement state and coherence state bits

```mermaid
graph TD
    subgraph "Cache Structure"
        CL[Cache Line] --> T[Tag]
        CL --> D[Data]
        CL --> S[Metadata]
        S --> R[Replacement State]
        S --> C[Coherence State]
    end
```

### 3.1.2 Address Mapping

- **Physical Address Components:**
	- Tag bits: Identify unique cache line within a set
	- Index bits: Determine cache set
	- Offset bits: Location within cache line
- **Example:** For a 4-way set-associative cache with 64-byte lines and 256 sets:
	- 6 bits offset (64 bytes = 2^6)
	- 8 bits index (256 sets = 2^8)
	- Remaining bits form the tag

![](Work@NV/Security/cachemap.svg)

## 3.2 Cache Hierarchy

### 3.2.1 Level Organization
1. **L1 Cache:**
	- Split into instruction (L1I) and data (L1D) caches
	- Smallest (32-64KB) but fastest (2-4 cycles)
	- Private to each CPU core

2. **L2 Cache:**
	- Unified (instructions and data)
	- Larger (256KB-1MB) but slower (~12 cycles)
	- Usually private per core

3. **Last Level Cache (L3/LLC):**
	- Largest (several MB to tens of MB)
	- Shared among all cores
	- Slower (~40-50 cycles) but prevents memory access (~200-300 cycles)

```mermaid
graph TD
    Core1[CPU Core 1] --> L1I1[L1I Cache]
    Core1 --> L1D1[L1D Cache]
    L1I1 --> L21[L2 Cache]
    L1D1 --> L21

    Core2[CPU Core 2] --> L1I2[L1I Cache]
    Core2 --> L1D2[L1D Cache]
    L1I2 --> L22[L2 Cache]
    L1D2 --> L22

    L21 --> LLC[L3/LLC Cache]
    L22 --> LLC
    LLC --> RAM[Main Memory]
```

## 3.3 Inclusion Policies

### 3.3.1 Inclusive Caches

- Lower level caches contain all data from upper levels
  - **Advantages:**
    - Simplified coherence management
    - Easy cross-core invalidation
  - **Disadvantages:**
    - Duplicate data reduces effective cache capacity
    - Evictions in LLC force evictions in L1/L2

### 3.3.2 Exclusive Caches

- Each cache line exists in only one cache level
	- **Advantages:**
		- Maximum effective cache capacity
		- No duplicate data
	- **Disadvantages:**
		- More complex coherence protocol
		- Higher migration overhead

### 3.3.3 Non-Inclusive Non-Exclusive (NINE)

- No strict inclusion/exclusion guarantee
- Common in modern processors
- Balances advantages of both approaches

```mermaid
graph TD
    subgraph "Cache Inclusion Policies"
        I[Inclusive] -->|"Contains"| IL1[L1 Data]
        I -->|"Contains"| IL2[L2 Data]

        E[Exclusive] -->|"Distinct"| EL1[L1 Data]
        E -->|"Distinct"| EL2[L2 Data]

        N[NINE] -->|"May Overlap"| NL1[L1 Data]
        N -->|"May Overlap"| NL2[L2 Data]
    end
```

## 3.4 Cache Buffers and Queues

### 3.4.1 Fill Buffers

- Handle cache line fills from lower levels
- Queue pending memory requests
- Can be exploited for timing attacks

### 3.4.2 Write Buffers

- Store pending writes to lower levels
- Implement write-back policies
- Can create timing variations

### 3.4.3 Victim Buffers

- Store recently evicted cache lines
- Reduce impact of conflict misses
- Additional timing channel source

### 3.4.4 Line Fill Buffers (LFB)

- Manage outstanding cache misses
- Track in-flight memory requests
- Critical for memory level parallelism

## 3.5 Replacement Policies

### 3.5.1 Common Replacement Policies

- **Least Recently Used (LRU)**
- **Pseudo-LRU (PLRU)**
- **Not Recently Used (NRU)**
- **Random Replacement**

## 3.6 Cache Coherence

### 3.6.1 MESI Protocol

Most common coherence protocol in modern processors:

- **Modified (M):** Line is dirty and exclusive to one core
- **Exclusive (E):** Line is clean and exclusive to one core
- **Shared (S):** Line may be present in multiple caches
- **Invalid (I):** Line is not present in cache

### 3.7 Cache Banking

- **Multi-banking:**
	- Divides cache into independent banks
	- Allows parallel access
	- Reduces port contention

- **Interleaving:**
	- Distributes addresses across banks
	- Improves access parallelism
	- Can create timing channels

# 4 Taxonomy of Cache Timing Attacks

| Requirement Type                   | **Tag-Based Attacks**                                                                                 |                                                 |                                                     |                                                                              |                                                                                                            |                                                            |                                                                   |                                                                          | **Metadata-Based Attacks**                                         | **Contention-Based Attacks**                               |                                                            |
| ---------------------------------- | ----------------------------------------------------------------------------------------------------- | ----------------------------------------------- | --------------------------------------------------- | ---------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------- | ----------------------------------------------------------------- | ------------------------------------------------------------------------ | ------------------------------------------------------------------ | ---------------------------------------------------------- | ---------------------------------------------------------- |
| **Category**                       | **Control-Based**                                                                                     |                                                 | **Conflict-Based**                                  |                                                                              |                                                                                                            |                                                            |                                                                   |                                                                          | **Coherence**                                                      | **Contention**                                             |                                                            |
| **Attack Name**                    | [Flush+Reload](https://www.usenix.org/system/files/conference/usenixsecurity14/sec14-paper-yarom.pdf) | [Flush+Flush](https://arxiv.org/abs/1511.04594) | [Prime+Probe](https://eprint.iacr.org/2005/271.pdf) | [Evict+Time](https://www.cs.tau.ac.il/~tromer/papers/cache-joc-20090619.pdf) | [Evict+Reload](https://www.usenix.org/system/files/conference/usenixsecurity15/sec15-paper-gruss-file.pdf) | [LRU Replacement Attack](https://arxiv.org/abs/1905.08348) | [Prime+Scope](https://dl.acm.org/doi/abs/10.1145/3466752.3480093) | [Reload+Refresh](https://www.usenix.org/system/files/sec19-briongos.pdf) | [Coherence Attack](https://dl.acm.org/doi/10.1145/2897937.2897944) | [CacheBleed](https://ieeexplore.ieee.org/document/7546524) | [WB Channel](https://ieeexplore.ieee.org/document/9407180) |
| **Description**                    | Leak data by measuring cache hit/miss                                                                 | Timing flush operation                          | Fill sets with attacker data                        | Observe victim timing                                                        | Evict+reload without flush                                                                                 | Exploit LRU replacement policy                             | Granular access timing                                            | Exploits cache replacement policies without forcing evictions            | Exploit coherence states                                           | Leak via port contention                                   | Exploit dirty bit writeback                                |
| **Target Applications**            | GnuPG, AES, keystrokes                                                                                | GnuPG, AES, keystrokes                          | GnuPG, AES                                          | AES                                                                          | GnuPG, AES                                                                                                 | GnuPG, AES                                                 | AES, RSA                                                          | AES, RSA                                                                 | AES                                                                | Unknown                                                    | GnuPG, AES                                                 |
| **Hardware Requirements**          |                                                                                                       |                                                 |                                                     |                                                                              |                                                                                                            |                                                            |                                                                   |                                                                          |                                                                    |                                                            |                                                            |
| Cache Level (L1)                   |                                                                                                       |                                                 |                                                     |                                                                              |                                                                                                            |                                                            |                                                                   |                                                                          |                                                                    | L1 (Can be used to LLC, but harder)                        |                                                            |
| Cache Banking                      |                                                                                                       |                                                 |                                                     |                                                                              |                                                                                                            |                                                            |                                                                   |                                                                          |                                                                    | ✓                                                          |                                                            |
| Inclusive Cache                    | ✓                                                                                                     | ✓                                               | ✓                                                   |                                                                              | ✓                                                                                                          |                                                            | ✓                                                                 | ✓                                                                        | ✓                                                                  |                                                            |                                                            |
| Cache Coherence                    |                                                                                                       |                                                 |                                                     |                                                                              |                                                                                                            |                                                            |                                                                   |                                                                          | ✓                                                                  |                                                            |                                                            |
| Replacement Policy                 |                                                                                                       |                                                 | ✓                                                   |                                                                              |                                                                                                            | ✓                                                          | ✓                                                                 | ✓                                                                        |                                                                    |                                                            | ✓                                                          |
| SMT Support                        |                                                                                                       |                                                 |                                                     |                                                                              |                                                                                                            | ✓                                                          |                                                                   |                                                                          |                                                                    | ✓                                                          |                                                            |
| **Software Requirements**          |                                                                                                       |                                                 |                                                     |                                                                              |                                                                                                            |                                                            |                                                                   |                                                                          |                                                                    |                                                            |                                                            |
| High-Precision Timer               | ✓                                                                                                     | ✓                                               | ✓                                                   | ✓                                                                            | ✓                                                                                                          | ✓                                                          | ✓                                                                 | ✓                                                                        | ✓                                                                  | ✓                                                          | ✓                                                          |
| Shared Memory                      | ✓                                                                                                     | ✓                                               |                                                     |                                                                              | ✓                                                                                                          |                                                            |                                                                   | ✓                                                                        | ✓                                                                  |                                                            |                                                            |
| Eviction Set                       |                                                                                                       |                                                 | ✓                                                   | ✓                                                                            | ✓                                                                                                          | ✓                                                          | ✓                                                                 | ✓                                                                        |                                                                    |                                                            | ✓                                                          |
| clflush Privilege                  | ✓                                                                                                     | ✓                                               |                                                     |                                                                              |                                                                                                            |                                                            |                                                                   |                                                                          |                                                                    |                                                            |                                                            |
| sw prefetch Privilege              |                                                                                                       |                                                 |                                                     |                                                                              |                                                                                                            |                                                            |                                                                   |                                                                          | ✓                                                                  |                                                            |                                                            |
| **Timing Channel Characteristics** |                                                                                                       |                                                 |                                                     |                                                                              |                                                                                                            |                                                            |                                                                   |                                                                          |                                                                    |                                                            |                                                            |
| Time Resolution                    | ~100 cycles                                                                                           | ~50 cycles                                      | ~100 cycles                                         | ~200 cycles                                                                  | ~100 cycles                                                                                                | Clean: 10 cycles<br>Dirty: 1300 cycles                     | ~100 cycles                                                       | ~100 cycles                                                              | ~26 cycles                                                         | ~10-20 cycles                                              | ~10 cycles                                                 |
| Bandwidth                          | 75-180 KB/s                                                                                           | ~100 KB/s                                       | 50-150 KB/s                                         | 20-50 KB/s                                                                   | 60-180 KB/s                                                                                                | 72.5 KB/s                                                  | 70-150 KB/s                                                       | 60-120 KB/s                                                              | 87.5-137.5 KB/s                                                    | Unknown                                                    | 1.3-4.4 KB/s                                               |
| Error Rate                         | <1%                                                                                                   | <5%                                             | 5-15%                                               | 10-20%                                                                       | <10%                                                                                                       | 8%                                                         | <10%                                                              | <8%                                                                      | <10%                                                               | Unknown                                                    | <8%                                                        |
| Channel Type                       | Hit/Miss                                                                                              | Flush Latency                                   | Hit/Miss                                            | Hit/Miss                                                                     | Hit/Miss                                                                                                   | Replacement policy                                         | Hit/Miss                                                          | Replace without eviction                                                 | State Transition                                                   | Port contention                                            | Replace latency                                            |

Cache timing attacks can be organized into three major classes based on the fundamental mechanisms they exploit:

1. **Tag-Based Attacks**: Exploit the tag comparison mechanism to determine whether data is present in the cache
   - **Control-Based Techniques**: Use explicit instructions to manipulate cache state
   - **Conflict-Based Techniques**: Create conflicts between cache lines (including replacement policy attacks)
2. **Metadata-Based Attacks**: Exploit auxiliary information maintained by caches beyond simple presence/absence of data
   - **Coherence-Based Attacks**: Exploit cache coherence protocols and their state transitions
3. **Contention-Based Attacks**: Exploit contention for shared resources in the cache system

## 4.1 Tag-Based Attacks

Tag-based attacks focus on the basic cache operation of determining whether data is present in the cache by manipulating the tag comparison mechanism. These attacks can be further divided based on how they manipulate cache state.

### 4.1.1 Control-Based Techniques

Control-based techniques use explicit CPU instructions (like flush or prefetch) to directly manipulate cache state, offering more precise control but potentially requiring specific privileges.

#### 4.1.1.1 Flush+Reload

- **Attack Requirements:**
	- Shared memory between attacker and victim
	- Access to flush instructions (e.g., `clflush` on x86)
	- Deterministic access patterns by the victim
	- Sufficient time between flush and reload for victim activity

- **Attack Flow:**
	1. **Flush:** The attacker flushes a targeted cache line (shared with the victim).
	2. **Wait:** The attacker waits for a short interval during which the victim may (or may not) access the flushed line.
	3. **Reload:** The attacker reloads that line and measures the latency (fast = victim accessed, slow = no victim access).

- **Key Characteristics:** Offers high accuracy and low noise but requires both flush privileges and shared data.

```mermaid
sequenceDiagram
    participant Attacker
    participant Cache
    participant Victim

    Attacker->>Cache: clflush(shared_line)
    Note right of Attacker: Wait interval
    Victim->>Cache: Possibly access shared_line
    Attacker->>Cache: Reload shared_line
    Attacker->>Attacker: Measure latency
    Note left of Attacker: Fast = victim accessed, Slow = no access
```

#### 4.1.1.2 Flush+Flush

- **Attack Requirements:**
	- Shared memory between attacker and victim
	- Access to flush instructions

- **Attack Flow:**
	1. The attacker issues continuous flush operations on a shared cache line.
	2. The attacker measures the time each flush takes.
	3. If the flush time is long, the line was cached (victim accessed it); if short, it was not cached.

- **Key Characteristics:** Higher temporal resolution and stealthier than Flush+Reload as it doesn't trigger cache misses.

```mermaid
sequenceDiagram
    participant Attacker
    participant Cache
    participant Victim

    Attacker->>Cache: clflush(shared_line)
    Attacker->>Attacker: Measure flush time
    Victim->>Cache: Possibly access shared_line
    Attacker->>Cache: clflush(shared_line)
    Attacker->>Attacker: Measure flush time again
    Note left of Attacker: Longer flush time -> victim accessed line
```

**Why Flush Timing Differs:**

```mermaid
graph TD
    subgraph "Flush Timing Difference"
        A[Flush uncached line] -->|"Direct invalidation<br>No coherence traffic"| B[Fast Operation]
        C[Flush cached line] -->|"Invalidate in cache<br>Update coherence state<br>Write-back if dirty"| D[Slower Operation]
    end
```

#### 4.1.1.3 Leaky Way (NTP+NTP)

- Leverages Intel `PREFETCHNTA` in a covert channel without needing a full prime or flush step before each measurement.
- Demonstrates how specialized instructions may yield robust channels with moderate to high bandwidth, disregarding conventional set-associative defenses.

```mermaid
sequenceDiagram
    participant Sender
    participant Cache
    participant Receiver

    Note over Sender,Receiver: Leaky Way (NTP+NTP)
    Sender->>Cache: PREFETCHNTA(shared_line) [bit=1]
    Note right of Sender: Or skip prefetch [bit=0]
    Receiver->>Cache: PREFETCHNTA(shared_line)
    Receiver->>Receiver: Measure prefetch latency
    Note left of Receiver: Fast = no previous prefetch (bit=0)<br>Slow = previous prefetch (bit=1)
```

### 4.1.2 Conflict-Based Techniques

Conflict-based techniques rely on creating cache conflicts to force target addresses out of the cache. They often require more preparation but can work without special privileges or shared memory.

#### 4.1.2.1 Prime+Probe
- **Attack Requirements:**
	- Knowledge of cache indexing function and cache replacement policy
	- Ability to construct effective eviction sets
	- Functional in cross-VM settings with shared LLC

- **Attack Flow:**
	1. **Prime:** Attacker fills a target cache set with its own data (evicting any victim lines).
	2. **Wait:** Victim may evict some attacker lines if it accesses data that maps to the same set.
	3. **Probe:** Attacker re-accesses its data in that set; slow accesses imply evictions caused by the victim.

- **Key Characteristics:** More widely applicable but typically noisier than methods requiring shared memory.

```mermaid
sequenceDiagram
    participant Attacker
    participant Cache
    participant Victim

    Attacker->>Cache: Access eviction set (prime the cache)
    Note right of Attacker: Wait
    Victim->>Cache: Possibly access lines in that set
    Attacker->>Cache: Probe set again
    Attacker->>Attacker: Measure latency (miss/hit)
```

#### 4.1.2.2 Evict+Time

- **Attack Requirements:**
	- Ability to trigger victim execution or measure total execution time
	- Knowledge of which cache sets to target
	- Effective eviction set construction

- **Attack Flow:**
	1. The attacker evicts specific lines the victim might use.
	2. The attacker triggers or waits for the victim's code to run.
	3. The attacker measures the victim's execution time; if the victim needed evicted lines, that code path becomes slower.

- **Key Characteristics:** Uses execution time as a side channel rather than direct cache access timings.

```mermaid
sequenceDiagram
    participant Attacker
    participant Cache
    participant Victim

    Attacker->>Cache: Evict lines victim might use
    Attacker->>Victim: Trigger victim execution
    Victim->>Cache: Possibly load evicted lines
    Attacker->>Attacker: Measure victim run time
    Note left of Attacker: Slow run = victim used evicted data
```

#### 4.1.2.3 Evict+Reload

- **Attack Requirements:**
	- Shared memory between attacker and victim
	- Effective eviction set construction
	- Precise cache set knowledge

- **Attack Flow:**
	1. The attacker constructs an eviction set that maps to the same cache set as the victim's data.
	2. The attacker accesses all addresses in the eviction set to evict the victim's line.
	3. The attacker waits; the victim may reload that line.
	4. The attacker reloads the same line and measures latency—fast indicates the victim used it, slow otherwise.

- **Key Characteristics:** Similar to Flush+Reload but uses conflict-based eviction rather than explicit flush instructions.

```mermaid
sequenceDiagram
    participant Attacker
    participant Cache
    participant Victim

    Attacker->>Attacker: Build eviction set
    Attacker->>Cache: Evict victim_line (via eviction set)
    Note right of Attacker: Wait interval
    Victim->>Cache: Possibly reload victim_line
    Attacker->>Cache: Reload victim_line
    Attacker->>Attacker: Measure latency
    Note left of Attacker: Fast = victim accessed
```

#### 4.1.2.4 Occupancy Attacks

- **Attack Requirements:**
	- Large buffer for probing total cache usage
	- High-precision timer for measuring aggregate access times
	- Distinctive victim cache usage patterns
	- Works in cross-VM settings due to shared LLC
	- Effective with limited knowledge of victim code structure

- **Attack Flow:**
	  1. The attacker establishes a baseline for accessing a large buffer that fills most of the cache.
	  2. During victim activity, the attacker repeatedly accesses this buffer and measures timing.
	  3. Changes in access time correlate with victim's cache usage.

- **Key Characteristics:** Does not require identifying exact victim addresses, measures overall cache footprint.

```mermaid
sequenceDiagram
    participant Attacker
    participant Cache
    participant Victim

    Note over Attacker,Victim: Occupancy Attack
    Attacker->>Attacker: Establish baseline cache access time
    Victim->>Cache: Run target operations
    Attacker->>Cache: Access large buffer repeatedly
    Attacker->>Attacker: Detect slowdown in buffer access
    Note left of Attacker: Higher slowdown = more cache occupancy by victim
```


#### 4.1.2.5 Prime+Scope

- **Attack Requirements:**
	- Eviction set construction ability
	- Knowledge of cache replacement policy
	- Inclusive cache architecture

- **Attack Flow:**
	1. **PRIME Phase:** The attacker accesses addresses mapping to the target cache set, filling the set with attacker-controlled data.
	2. **Wait Phase:** The attacker waits for the victim to potentially access data mapping to the same cache set.
	3. **SCOPE Phase:**
	 - The attacker accesses each cache line in sequence and measures individual access times rather than aggregate time.
	 - By analyzing the pattern of access times, the attacker can determine which specific ways were evicted by the victim.

- **Key Characteristics:** Higher spatial resolution than traditional Prime+Probe, providing more detailed information about victim accesses.

```mermaid
sequenceDiagram
    participant Attacker
    participant Cache
    participant Victim

    Note over Attacker,Victim: PRIME Phase
    Attacker->>Cache: Access addresses in target cache set
    Note right of Attacker: Fill cache set with attacker's data

    Note over Attacker,Victim: Wait for Victim
    Note right of Attacker: Victim may access data in same cache set

    Note over Attacker,Victim: SCOPE Phase
    Attacker->>Cache: Re-access first cache line in sequence
    Attacker->>Attacker: Measure access time (T1)
    Note left of Attacker: This line should still be in cache

    Attacker->>Cache: Access rest of contention set in sequence
    Attacker->>Attacker: Measure access times (T2...Tn)

    Note over Attacker,Victim: Analysis
    Note left of Attacker: Compare timing patterns
    alt Victim accessed cache set
        Note left of Attacker: Some lines evicted, creating timing gaps
    else Victim didn't access cache set
        Note left of Attacker: All lines still in cache, uniform timing
    end
```

#### 4.1.2.6 Reload+Refresh

- **Attack Requirements:**
	- Shared memory between attacker and victim
	- Predictable cache replacement policy implementation
	- Inclusive cache hierarchy
	- Ability to manipulate cache replacement state

- **Attack Flow:**
	1. **Setup Phase:** The attacker fills a cache set with the target address and all but one member of an eviction set, with all cache lines set to medium freshness (age=2).
	2. **Observation Phase:** If the victim accesses the target, its age value is updated to fresh (age=1); otherwise, it remains at age=2.
	3. **Trigger Phase:** Loading the final eviction set member forces replacement - if the target was accessed (age=1), it stays in cache; if not (age=2), it gets evicted.
	4. **Detection Phase:** Reload time reveals whether the target remained in cache (fast) or was evicted (slow).
	5. **Reset Phase:** Restore initial cache state for the next observation round.

- **Key Characteristics:** Doesn't force evictions of victim's data during observation, making it stealthier against certain detection mechanisms.

```mermaid
sequenceDiagram
    participant Attacker
    participant Cache
    participant Victim

    Note over Attacker,Victim: Initial Setup
    Attacker->>Cache: Load target address and eviction set members [except ev_{w-1}]
    Note right of Attacker: All cache lines set with age=2

    Note over Attacker,Victim: RELOAD Phase (Observation)
    Note right of Attacker: Wait for victim activity
    alt Victim accesses target address
        Victim->>Cache: Access target address
        Note left of Cache: Target age = 1 (fresh)
    else Victim doesn't access target
        Note left of Cache: Target age remains 2
    end

    Note over Attacker,Victim: Trigger eviction
    Attacker->>Cache: Load ev_{w-1} (causes eviction)
    alt Target was accessed (age=1)
        Note left of Cache: Target stays in cache, other lines age=3
    else Target was not accessed (age=2)
        Note left of Cache: Target gets evicted, ev_{w-1} enters cache
    end

    Note over Attacker,Victim: Detection
    Attacker->>Cache: Reload target address
    Attacker->>Attacker: Measure access time
    alt Fast access
        Note left of Attacker: Target was in cache (victim accessed it)
    else Slow access
        Note left of Attacker: Target was evicted (victim didn't access it)
    end

    Note over Attacker,Victim: REFRESH Phase (Reset)
    Attacker->>Cache: Flush ev_{w-1}
    Attacker->>Cache: Reload target address (age=2)
    Attacker->>Cache: Load remaining eviction set to restore initial state
```

#### 4.1.2.7 LRU State Attack

- **Attack Requirements:**
	- CPU with predictable LRU or pseudo-LRU cache replacement policy
	- Simultaneous Multi-Threading (SMT) support

- **Attack Flow (Shared Memory Scenario):**
	1. **Initialization:** Establish known LRU order in target cache set by accessing cache lines in a specific sequence.
	2. **Transmission:** Sender accesses (or doesn't access) specific shared memory addresses to modify LRU states.
	3. **Detection:** Receiver measures access times for all cache lines to detect LRU state changes.
	4. **Reset:** Re-establish initial LRU state for the next round.

- **Key Characteristics:**
  - Exploits subtle timing variations without requiring cache misses
  - Works in both shared and non-shared memory scenarios
  - High transmission rates (up to 600 Kbps per cache set)
  - Can bypass secure cache designs focused only on preventing traditional cache attacks

```mermaid
sequenceDiagram
    participant Attacker
    participant Cache
    participant Victim

    Note over Attacker,Victim: Initialization Phase
    Attacker->>Cache: Access cache lines in specific sequence
    Note right of Attacker: Establish known LRU order

    Note over Attacker,Victim: Transmission Phase
    alt Transmit bit 1
        Victim->>Cache: Access target cache line
        Note left of Cache: Modifies LRU state (line becomes MRU)
    else Transmit bit 0
        Note left of Victim: No access
        Note left of Cache: LRU state remains unchanged
    end

    Note over Attacker,Victim: Detection Phase
    Attacker->>Cache: Access cache lines in sequence
    Attacker->>Attacker: Measure access times
    Note left of Attacker: Analyze timing patterns to detect LRU state changes

    Note over Attacker,Victim: Reset Phase
    Attacker->>Cache: Re-access cache lines in specific order
    Note right of Attacker: Restore known LRU state for next round

    Note over Attacker,Victim: Result
    alt Detected LRU state change
        Note left of Attacker: Infer bit 1 was transmitted
    else No LRU state change detected
        Note left of Attacker: Infer bit 0 was transmitted
    end
```

## 4.2 Metadata-Based Attacks

Metadata-based attacks exploit auxiliary information maintained by cache systems beyond simple presence/absence of data. These attacks target the coherence protocols, replacement policies, and other metadata structures maintained by modern cache hierarchies.

### 4.2.1 Coherence-State Attacks

#### 4.2.1.1 Reload (Core A) + Reload (Core B)

- **Attack Requirements:**
	- Multi-core processor architecture
	- Cache coherence protocol (e.g., MESI or variants)
	- Shared memory between attacker processes
	- Ability to force specific coherence states
	- High-precision timer for detecting state transitions
	- Cross-core communication via shared cache

- **Attack Flow:**
	1. A trojan process ensures a shared cache line is in Exclusive (E) or Modified (M) state on one core.
	2. The victim may access that line, forcing a coherence state transition (e.g., E->S).
	3. A spy process on another core accesses the same line and measures additional latency from state transitions.

- **Key Characteristics:** Exploits coherence protocol timing differences rather than simple presence/absence of data.

```mermaid
sequenceDiagram
    participant Trojan
    participant Victim
    participant Spy
    participant Cache

    Trojan->>Cache: Place line in E/M state
    Note right of Trojan: Wait interval
    Victim->>Cache: Access line -> E->S transition
    Spy->>Cache: Access same line
    Spy->>Spy: Measure latency (fast vs. slow)
    Note left of Spy: Fast = line was already S, Slow = still E or M
```


#### 4.2.1.2 Prefetch+Reload

- **Attack Requirements:**
	- Access to software prefetch instructions
	- Shared memory between attacker and victim

- **Attack Flow:**
	1. The attacker prefetches a shared line into the cache.
	2. The attacker waits while the victim may or may not access the line.
	3. The attacker reloads (or re-prefetches) and measures the latency.

- **Key Characteristics:** Can circumvent certain flush restrictions while providing similar information leakage.

```mermaid
sequenceDiagram
    participant Attacker
    participant Cache
    participant Victim

    Attacker->>Cache: prefetch(shared_line)
    Note right of Attacker: Wait interval
    Victim->>Cache: Possibly access shared_line
    Attacker->>Cache: reload(shared_line)
    Attacker->>Attacker: Measure latency
    Note left of Attacker: Fast = victim accessed, Slow = no access
```


## 4.3 Contention-Based Attacks

Contention-based attacks exploit resource contention within cache structures, such as port contention in banked caches or timing differences in write-back operations.

### 4.3.1 CacheBleed

- **Attack Requirements:**
	- L1 cache with banked implementation
	- Simultaneous Multi-Threading (SMT) support
	- Knowledge of address-to-bank mapping
	- Victim code with address-dependent memory patterns
	- Processor allowing concurrent cache bank access

- **Attack Flow:**
	  1. The attacker creates controlled access patterns to specific cache banks.
	  2. By measuring timing variations when the victim is active, the attacker can infer victim's memory access patterns.
	  3. These patterns can reveal sensitive information like cryptographic keys.

- **Key Characteristics:** Exploits contention for cache banks rather than cache line presence/absence.

### 4.3.2 WB Channel (Write-Back Channel)

- **Attack Requirements:**
	- Cache architecture with write-back policy
	- Significant timing difference between clean and dirty line replacements
	- Eviction set construction capability

- **Attack Flow:**
	1. The receiver (spy) ensures the target cache line is initially clean.
	2. The sender (trojan) modifies that line, marking it dirty.
	3. The spy forces a replacement (or eviction) in that cache set.
	4. Checking the eviction latency reveals whether a dirty write-back occurred.

- **Key Characteristics:** Exploits the substantial timing difference between replacing clean vs. dirty cache lines. Timing difference is caused by writeback and replace require reading/writing the same cache line

```mermaid
sequenceDiagram
    participant Spy
    participant Trojan
    participant Cache

    Spy->>Cache: Ensure line is clean
    Trojan->>Cache: Write to line -> dirty state
    Spy->>Cache: Force replacement
    Spy->>Spy: Measure writeback latency
    Note left of Spy: Higher latency = dirty line was replaced
```

```mermaid
graph TD
    subgraph "Replacement Policy Attack"
        Normal[Normal Eviction] -->|"Fast eviction time"| CleanLine[Clean Line]
        HighLatency[High Eviction Latency] -->|"Indicates"| DirtyLine[Dirty Line]
        DirtyLine -->|"Requires write-back to memory"| HighLatency
    end
```

# 5 Timing Channel Characteristics

Cache timing channels exhibit distinct characteristics in terms of resolution, bandwidth, and error rates. These metrics vary significantly based on the specific attack technique and cache level targeted.

| Channel Type              | Time Resolution                                                      | Bandwidth (bps) | Bit Error Rate | Key Characteristics                                                                                                |
| ------------------------- | -------------------------------------------------------------------- | --------------- | -------------- | ------------------------------------------------------------------------------------------------------------------ |
| Cache Hit/Miss            | ~100 cycles                                                          | 75-180          | <1%            | - Most reliable timing channel<br>- Direct measurement of cache state<br>- Affected by system noise and scheduling |
| Cache Directory Coherence | 26 cycles (local)                                                    | 87.5-137.5      | <10%           | - Cross-core communication<br>- Coherence protocol overhead<br>- State transition timing                           |
| Replacement States        | 5-15 cycles                                                          | 72.5            | 8%             | - Metadata-based timing<br>- Replacement policy dependent<br>- Lower bandwidth but stealthier                      |
| L1 Cache Port Contention  | 10-20 cycles                                                         | N/A             | N/A            | - Resource contention based<br>- Highly dependent on workload<br>- Can be noisy                                    |
| Replacement Latency       | Clean: ~10 cycles<br>Dirty: ~1300 cycles<br>Write-back: ~4400 cycles | N/A             | <8%            | - Large timing difference<br>- Write-back dependent<br>- High reliability                                          |

# 6 Attack Requirements and Practical Considerations

## 6.1 High-Precision Timing Sources

- **Hardware Timers:** Intel `rdtsc`, AMD `APERF`, ARM `PMCCNTR` (often privileged).
- **Performance Counters:** High accuracy but frequently restricted to kernel use.
- **Software Timers (Counting Threads):** Easier to implement, but typically lower resolution and can be subject to scheduling noise.

```mermaid
graph LR
      subgraph "Timing Sources"
          HW[Hardware Timers] -->|"High precision<br>May be restricted"| Timing
          PC[Performance Counters] -->|"High precision<br>Often privileged"| Timing
          SW[Software Timers] -->|"Lower precision<br>Less restrictions"| Timing
          Timing[Timing Measurement]
      end
```

## 6.2 Privilege Requirements

- **Flush Instructions:** Commonly available on Intel x86 but can be restricted on other architectures.
- **Eviction Set Construction:** Requires knowledge of virtual-physical-address mappings or large-scale scanning.
- **Shared Memory:** Some attacks (e.g., Flush+Reload, Evict+Reload) need attacker-victim page sharing; others (Prime+Probe) do not.

```mermaid
graph TD
      subgraph "Attack Privilege Requirements"
          ControlBased[Control-Based] -->|"Often requires"| Flush[Flush Instruction]
          ControlBased -->|"Often requires"| Shared[Shared Memory]

          ConflictBased[Conflict-Based] -->|"Requires"| Eviction[Eviction Set Construction]
          ConflictBased -->|"May not require"| Shared

          MetadataBased[Metadata-Based] -->|"May require"| Shared
          MetadataBased -->|"May require"| SpecialInst[Special Instructions]
      end
```

## 6.3 Cross-Core vs. Same-Core

- **Same-Core Attacks:** Exploit shared L1 caches and shared pipeline resources, often achieving higher bandwidth.
- **Cross-Core Attacks:** Rely on shared LLC or bus coherence, sometimes sacrificing bandwidth but broadening applicability.

```mermaid
graph LR
      subgraph "Attack Scope"
          SC[Same-Core Attacks] -->|"Higher bandwidth<br>L1/L2 accessible"| Scope
          CC[Cross-Core Attacks] -->|"Broader reach<br>LLC only"| Scope
          Scope[Attack Scope]
      end
```

## 6.4 Shared Memory Generation

Obtaining shared memory between attacker and victim processes is crucial for several attack types. Here are the primary methods:

- **Shared Libraries:**
	- **Mechanism:** Both victim and attacker load the same shared library code (e.g., libc).
	- **Exploitation:** The attacker can monitor victim access patterns to shared library functions.
	- **Example:** Cryptographic library accesses to lookup tables during encryption.

- **Memory Deduplication:**
	- **Mechanism:** Operating systems (e.g., Linux KSM, Windows TPS) merge identical memory pages to save RAM.
	- **Exploitation:** The attacker creates pages identical to victim's, allowing the OS to merge them.
	- **Process:**
		1. Attacker analyzes victim binary/libraries to identify target memory pages.
		2. Attacker creates identical pages in its own memory space.
		3. Wait for deduplication to merge the pages.
		4. Now both processes share physical memory while having different virtual addresses.

- **Page Table Manipulation (requires privileges):**
	- **Mechanism:** Direct mapping of attacker pages to victim's physical memory.
	- **Requirements:** Usually needs elevated privileges (e.g., `/proc/*/pagemap` access in Linux).

- **Memory-Mapped Files:**
	- **Mechanism:** Both processes map the same file into memory.
	- **Example:** Shared objects, configuration files, or world-readable files.


## 6.5 Eviction Set Construction

Building an effective eviction set is critical for conflict-based attacks. An eviction set is a collection of memory addresses that map to the same cache set as the target address.

- **Prerequisites for Construction:**
	- Understanding of cache organization (sets, ways, indexing)
	- Knowledge of how virtual addresses map to physical addresses and cache sets

- **Traditional Construction Methods:**
	1. **Group Testing (most common):**
		 - **Process:** Start with a large pool of addresses, repeatedly test subsets to find conflicting addresses.
		 - **Algorithm:**
			- Generate thousands of candidate addresses.
			- Test access latency after accessing candidate subset.
			- Iteratively refine candidates to find minimal conflicting set.

  2. **Address Bit Analysis:**
     - **Process:** Exploit knowledge of which address bits determine cache set index.
     - **Requirements:** Needs information about cache organization and address translation.

- **Modern Optimization Techniques:**
	  1. **Prime+Abort:** Uses Intel TSX (Transactional Memory) to detect eviction-caused aborts.
	  2. **Statistical Methods:** Correlation-based techniques to identify conflicting addresses.
	  3. **Dynamic Profiling:** Runtime profiling to identify optimal eviction sets.

- **Overcoming Cache Complexities:**
	- **Sliced Caches:** Modern processors often use sliced LLCs where addresses are distributed across slices.
	- **Solution:** Additional analysis required to determine slice mapping functions.
- **Practical Performance:**
	- **Traditional Methods:** Minutes to construct reliable eviction sets.
	- **Optimized Techniques:** Seconds or less on modern systems.
	- **Set Size:** Typically requires W+1 addresses (where W is cache associativity) for reliable eviction.

```mermaid
graph TD
      subgraph "Eviction Set Construction"
          GEN[Generate Large Pool<br>of Candidate Addresses] --> TEST[Test for Conflicts]
          TEST --> REFINE[Refine Candidates]
          REFINE -->|"Minimal set found"| FINAL[Final Eviction Set]
          REFINE -->|"Set incomplete"| TEST
      end

      subgraph "Testing Process"
          A[Access Candidate Set] --> B[Access Target Address]
          B --> C[Measure Access Time]
          C -->|"Long access time"| D[Conflict Detected]
          C -->|"Short access time"| E[No Conflict]
      end
```

# 7 Conclusion

This revised taxonomy of cache timing attacks provides a more fundamental organization based on the underlying mechanisms exploited. Tag-based attacks, subdivided into control-based and conflict-based techniques, manipulate the basic cache line presence detection mechanism, while metadata-based attacks target auxiliary information maintained by modern cache hierarchies.

Each attack category has its own requirements, advantages, and limitations:

- **Control-Based Techniques:** Offer high precision but often require specific instructions or privileges
- **Conflict-Based Techniques:** More widely applicable but may require complex preparation
- **Metadata-Based Attacks:** Exploit increasingly sophisticated cache designs, potentially bypassing defenses against traditional attacks

As cache architectures continue to evolve, we can expect attackers to find new ways to exploit both the fundamental mechanisms and the increasingly complex metadata structures of modern caches. Comprehensive defense strategies must therefore consider both basic tag-comparison mechanisms and the various auxiliary structures that support efficient cache operation.

```mermaid
graph TD
      subgraph "Defense Considerations by Attack Type"
          TBD[Tag-Based Defenses] -->|"Address"| CBT[Control-Based Techniques]
          TBD -->|"Address"| CoT[Conflict-Based Techniques]
          MBD[Metadata-Based Defenses] -->|"Address"| MA[Metadata Attacks]

          CBT -->|"Requires"| FRestrict[Flush Instruction Restrictions]
          CoT -->|"Requires"| CRand[Cache Randomization]
          MA -->|"Requires"| MSimp[Metadata Simplification]

          ALL[Comprehensive Defense] --> TBD
          ALL --> MBD
          ALL --> Perf[Performance Considerations]
      end
```
