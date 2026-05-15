# Realtime Violation Patterns and Safe Alternatives

Reference for the `audio-dsp-review` skill. The core principles here apply equally across
Windows (WASAPI, ASIO), macOS (CoreAudio, AudioUnit), Linux (ALSA, JACK), iOS, and Android.

## Table of Contents
1. [Memory Allocation](#1-memory-allocation)
2. [Locking and Priority Inversion](#2-locking-and-priority-inversion)
3. [System Calls and Blocking I/O](#3-system-calls-and-blocking-io)
4. [Poor Worst-Case Algorithms](#4-poor-worst-case-algorithms)
5. [Parameter Sharing Patterns](#5-parameter-sharing-patterns)
6. [Realtime-Safe Data Structures](#6-realtime-safe-data-structures)
7. [Framework-Specific Guidance](#7-framework-specific-guidance)
8. [Audit Checklist](#8-audit-checklist)

---

## 1. Memory Allocation

General-purpose allocation has no realtime bound. An allocator may take locks, grow or scan
arenas, request pages from the OS, trigger first-touch page faults, or run algorithms optimized
for average throughput rather than worst-case latency. An audio thread calling `malloc`/`new`
therefore risks blocking or running far longer than the buffer deadline.

**Common violations:**

```cpp
// BAD — string allocates
std::string name = "ch_" + std::to_string(idx);

// BAD — vector may reallocate when full
events.push_back(e);

// BAD — map inserts default value if key absent
auto& state = filterStates[voiceId];

// BAD — exception machinery may allocate
throw std::runtime_error("unexpected state");

// BAD — make_shared allocates ref-count block
auto node = std::make_shared<Node>();
```

**Safe alternatives:**

```cpp
// Pre-allocate in prepareToPlay / constructor
void prepareToPlay(double sr, int blockSize) {
    events.reserve(512);
    filterStates.reserve(maxVoices);  // or use std::array<FilterState, MAX_VOICES>
}

// Stack string — no allocation
char name[32];
snprintf(name, sizeof(name), "ch_%d", idx);

// Error codes, not exceptions
if (state < 0) return; // validate earlier, handle in UI thread

// Own the lifetime; pass raw ptr or reference
Node node;  // stack/member allocation
processNode(node);
```

---

## 2. Locking and Priority Inversion

A mutex is dangerous even when "almost never contended." Priority inversion: the audio thread
(high OS priority) blocks waiting for a mutex held by the UI thread (low priority). The OS
won't preempt the UI thread to help the audio thread — it just waits. Another thread at medium
priority can preempt the UI thread, further delaying the audio thread. Result: a dropout even
though the audio thread has the highest priority.

General-purpose OS schedulers do not guarantee priority inheritance. Even if they do (Linux
`PTHREAD_PRIO_INHERIT`), you're depending on undocumented scheduler behavior that can change
between OS releases.

**TryLock caveat:** `try_lock()` avoids blocking, but gives no guarantee you'll ever acquire the
lock — so you can't use it to protect anything that must be accessed every callback.

**Safe alternatives — prefer in this order:**

1. Lock-free `std::atomic<T>` for scalars — verify with `is_lock_free()` for the target type/platform
2. SPSC ring buffer for passing structs or events between UI↔audio
3. Double-buffering with atomic swap for larger parameter snapshots
4. `try_lock` only as a last resort, only for optional-access patterns

```cpp
// GOOD — atomic for parameter
std::atomic<float> gain{1.0f};
void setGain(float g) { gain.store(g, std::memory_order_relaxed); }
void processBlock(...) { applyGain(buf, gain.load(std::memory_order_relaxed)); }

// GOOD — SPSC queue for events
// e.g. juce::AbstractFifo, moodycamel::ReaderWriterQueue
```

---

## 3. System Calls and Blocking I/O

Any call that transitions to kernel mode can block. The terminal/log buffer may be full;
the disk head may need to seek; the socket may be waiting for data. Even a single `printf`
can stall for milliseconds.

**Logging — lock-free ring buffer pattern:**

```cpp
// Audio thread — no allocation, no I/O:
struct LogEntry { char msg[128]; };
juce::AbstractFifo logFifo{256};
std::array<LogEntry, 256> logBuf;

void audioLog(const char* m) {
    int start, n;
    logFifo.prepareToWrite(1, start, n);
    if (n > 0) { strncpy(logBuf[start].msg, m, 127); logFifo.finishedWrite(1); }
}

// Timer on message thread — drain and display:
void timerCallback() {
    int start, n;
    logFifo.prepareToRead(logFifo.getNumReady(), start, n);
    for (int i = start; i < start + n; i++) DBG(logBuf[i].msg);
    logFifo.finishedRead(n);
}
```

**File I/O:** Load files in a background thread. Use an `std::atomic<bool>` ready flag or an
SPSC queue to notify the audio thread when data is ready. Never open/read/write from the callback.

---

## 4. Poor Worst-Case Algorithms

Real-time requires good *worst-case* complexity, not average-case. An algorithm that runs fast
99.9% of the time but occasionally spikes 1000× will glitch. Examples:

- `memset` on a large delay line — takes O(N) time, may stall
- `std::map` lookup — O(log N), cache-hostile, heap-allocated nodes
- `std::unordered_map` — amortized O(1) but occasionally O(N) rehash
- Garbage collector pauses in managed runtimes (Java, C#)
- Virtual memory page faults when cold memory is first accessed

**Prefer:** O(1) worst-case algorithms. Pre-touch all buffers in `prepareToPlay` to warm pages.
Spread bursty computation across multiple callback invocations.

---

## 5. Parameter Sharing Patterns

The most common architecture bug: parameters written by the UI thread, read by the audio thread.

### Pattern A — `std::atomic<float>` (scalars)
```cpp
std::atomic<float> cutoff{1000.f}, resonance{0.7f};
std::atomic<bool> bypass{false};
// UI thread writes, audio thread reads — no lock needed
```

### Pattern B — SPSC queue (events / multi-field updates)
Use `juce::AbstractFifo`, `moodycamel::ReaderWriterQueue`, or similar. UI thread produces;
audio thread polls at the top of each callback.

### Pattern C — Double-buffer + atomic index swap
```cpp
struct Params { float cutoff, resonance; int algo; };
Params bufs[2];
std::atomic<int> pending{-1};
int active = 0;

// Audio thread:
int p = pending.exchange(-1, std::memory_order_acquire);
if (p >= 0) active = p;
const Params& p = bufs[active];
```

---

## 6. Realtime-Safe Data Structures

| Need | Safe option |
|------|-------------|
| Single scalar | Lock-free `std::atomic<T>`; verify `is_lock_free()` for non-integral types |
| 1-producer / 1-consumer queue | SPSC ring buffer (`juce::AbstractFifo`, `moodycamel::RWRQ`) |
| Multiple producers or consumers | MPMC lock-free queue (`moodycamel::ConcurrentQueue`) |
| Fixed-size buffer | `std::array<T, N>` (stack) |
| Variable-size, pre-allocated | `std::vector` with `reserve()` — never grow in callback |
| Fixed-capacity container | `etl::vector<T,N>`, `heapless::Vec<T,N>` (Rust) |
| Stack string | `std::array<char, N>` + `snprintf` |
| Circular audio buffer | Custom ring buffer or `juce::AbstractFifo` |

---

## 7. Framework-Specific Guidance

### JUCE
- `juce::String` — may allocate and may touch shared string machinery; never use in `processBlock`
- `DBG(x)` — calls `Logger::writeToLog()` which does I/O in debug builds; use lock-free log buffer
- `juce::AbstractFifo` — lock-free SPSC FIFO; use for audio→UI communication
- `AudioProcessorValueTreeState::getRawParameterValue()` — returns `std::atomic<float>*`, safe in callback
- `AudioBuffer<T>` — pre-allocated by host; never resize in callback
- `juce::SpinLock` — not OS-blocking, but burns CPU; prefer `std::atomic`

### VST3
- `IAudioProcessor::process()` must be fully allocation-free and lock-free
- Read parameter changes via `IParamValueQueue` (designed for polling)
- `IBStream` is for state save/restore only — never call from `process()`

### CLAP
- Same rules apply to the `process()` callback
- `clap_event_*` are host-owned; don't hold references after callback returns

### Rust / cpal
- Avoid `Vec::push`, `String` ops, `Box::new`, `Arc::clone` (clone is fine, allocating isn't)
- Use `heapless` crate for fixed-capacity collections
- Add `assert_no_alloc` crate in dev builds — panics if the closure allocates

### Web Audio API (AudioWorkletProcessor)
- `process()` runs on an isolated worklet thread — no DOM or `window`; load resources outside the callback
- Use `SharedArrayBuffer` + integer `Atomics` for lock-free communication with main thread
- Don't create new `Float32Array` inside `process()` — use the pre-allocated views
- Batch `postMessage` calls; don't call in every block

---

## 8. Audit Checklist

```
[ ] No new/delete/malloc/free or smart pointer creation
[ ] No std::string construction or concatenation
[ ] No container insertions that may grow heap
[ ] No std::mutex, lock_guard, unique_lock, scoped_lock
[ ] No condition_variable::wait(), future::get()
[ ] No printf/cout/DBG/syslog or any file I/O
[ ] No sleep/yield/nanosleep
[ ] No exception throw or catch
[ ] Parameters shared via std::atomic or lock-free queue
[ ] Buffers pre-allocated and pre-touched in prepareToPlay
[ ] Denormal protection in place (ScopedNoDenormals / _MM_SET_FLUSH_ZERO_MODE)
[ ] All called functions transitively satisfy the above
```
