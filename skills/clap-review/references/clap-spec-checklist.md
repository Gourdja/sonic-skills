# CLAP Plugin Spec Checklist

Full audit reference organized by extension. Check every applicable section against the actual implementation.

---

## Threading Model

CLAP defines three thread contexts. Every API entry point specifies which context it runs in.

| Context | Description | Examples |
|---------|-------------|---------|
| `[main-thread]` | Host's main/UI thread. Single-threaded relative to other main-thread calls. | `init`, `destroy`, `activate`, `deactivate`, GUI methods, state save/load, params flush when inactive |
| `[audio-thread]` | The realtime audio callback thread. Must be realtime-safe. | `process`, `start_processing`, `stop_processing` |
| `[thread-safe]` | May be called from any thread unless further restricted; implementation must be concurrency-safe. | `get_extension`, `host->request_callback`, thread-check calls |

**Golden rule:** obey the thread tag on the exact method. Never call a `[main-thread]`
host method from `[audio-thread]` context; use `host->request_callback()` to schedule
`plugin->on_main_thread()`.

---

## Core Plugin (`clap_plugin_t`)

| Check | Pass condition |
|-------|---------------|
| `init` queries only host extensions, stores results | No audio processing in `init` |
| `activate` allocates all runtime resources | Buffers, DSP objects, queues sized here |
| `deactivate` releases runtime resources | Symmetric with `activate` |
| `start_processing` / `stop_processing` are lightweight | No allocation; used for transport state only |
| `process` returns correct status | `CONTINUE`, `CONTINUE_IF_NOT_QUIET`, `TAIL`, or `SLEEP` — never an uninitialised value |
| `process` does not call any `[main-thread]` or `[!audio-thread]` host method directly | Emits output events or uses a main-thread callback request |
| All event types from `process->in_events->get(i)` are handled or explicitly ignored | Switch covers all known event types; default case is safe |
| Event pointers from `clap_input_events` are NOT stored beyond process() | Copied by value if needed later |
| Host queries plugin extensions only during or after `plugin->init()` | Never calls `plugin->get_extension()` before `init()` |

---

## Extension: `clap-plugin-audio-ports`

| Check | Pass condition |
|-------|---------------|
| Port count is stable after `init` | `count()` returns the same value for the plugin's lifetime (for a given config) |
| `get()` returns valid `clap_audio_port_info_t` for every index | No out-of-bounds; `id` is unique per port |
| Port `channel_count` matches actual buffer usage in `process()` | No reads/writes beyond declared channel count |
| `port_type` set correctly (`CLAP_PORT_STEREO`, `CLAP_PORT_MONO`, etc.) | Mismatched type confuses host channel routing |
| `in_place_pair` used only when plugin supports in-place processing | Only set when input and output buffer aliasing is handled |

**BAD:**
```c
// process() reads channel 2 on a declared mono port
float* left  = process->audio_inputs[0].data32[0];
float* right = process->audio_inputs[0].data32[1]; // UB: only 1 channel declared
```

**GOOD:**
```c
uint32_t ch_count = process->audio_inputs[0].channel_count;
for (uint32_t c = 0; c < ch_count; ++c) {
    float* ch = process->audio_inputs[0].data32[c];
    // ...
}
```

---

## Extension: `clap-plugin-params`

| Check | Pass condition |
|-------|---------------|
| `param_id` values are stable across plugin versions and save/load | Never derived from a container index or pointer |
| `count()` returns same count for a given plugin configuration | Count may vary with configuration but not randomly |
| `get_value()` is `[main-thread]` safe | Uses atomic read or main-thread copy |
| `value_to_text()` / `text_to_value()` do not allocate excessively | Stack buffers or pre-allocated scratch |
| Parameter changes from `clap_input_events` are applied in sample-accurate order | Sorted by `.time` within the block |
| `flush()` (called on main-thread when not processing) updates param state | Does not touch audio-thread-only state without a lock-free queue |
| `flush()` while active follows the spec thread context | May be called on the audio-thread when active, never concurrently with `process()` |
| Param changes from audio thread go through `output_events`, not direct mutation or `request_flush()` | Host reads output events to sync automation |

**BAD:**
```c
// Storing index as ID — breaks if plugin reorders params
info->id = index; // fragile
```

**GOOD:**
```c
// Stable ID from a fixed enum
info->id = PARAM_GAIN; // enum value, never changes
```

---

## Extension: `clap-plugin-state`

| Check | Pass condition |
|-------|---------------|
| `save()` runs entirely on `[main-thread]` | No audio thread access |
| `load()` runs entirely on `[main-thread]` | Applies state synchronously; signals audio thread via atomic/queue |
| `save()` does not do blocking I/O | Writes to `clap_ostream_t` only |
| `load()` validates data before applying | Gracefully handles truncated or versioned data |
| State format is versioned | Can read older plugin state without crashing |
| Saved param IDs match the current `clap-plugin-params` set | Mismatch causes silent parameter loss |

---

## Extension: `clap-plugin-gui`

| Check | Pass condition |
|-------|---------------|
| All GUI calls are on `[main-thread]` | Never call GUI methods from audio thread |
| `set_size()` / `get_size()` negotiate correctly with host | Plugin does not assume a fixed window size |
| `destroy()` cleans up all platform resources | No resource leak when editor is closed |
| GUI does not directly access audio-thread state | Uses atomics or message queues to read/write DSP params |
| `clap_gui_resize_hints_t` filled correctly | `can_resize_*` flags match actual resize behaviour |

---

## Extension: `clap-plugin-log`

| Check | Pass condition |
|-------|---------------|
| `host_log->log()` not called from `[audio-thread]` | Log from main thread only or buffer messages for later |
| Severity levels used correctly | `CLAP_LOG_ERROR` for errors, `DEBUG` for development noise |

---

## Extension: `clap-plugin-timer-support`

| Check | Pass condition |
|-------|---------------|
| `on_timer()` runs on `[main-thread]` | No audio state mutation without synchronisation |
| Timers unregistered in `deactivate` or `destroy` | No dangling timer IDs |
| Timer period is appropriate | UI refresh at 60 Hz max; parameter polling at 30 Hz typical |

---

## Common Mistake Patterns

### Event lifetime violation

**BAD:**
```c
// Storing pointer to a host-owned event
static const clap_event_note_t* last_note;

void process(...) {
    const clap_event_header_t* hdr = in_events->get(in_events, 0);
    last_note = (const clap_event_note_t*)hdr; // use-after-free after process() returns
}
```

**GOOD:**
```c
// Copy by value before process() returns
static clap_event_note_t last_note_copy;

void process(...) {
    const clap_event_header_t* hdr = in_events->get(in_events, 0);
    if (hdr->type == CLAP_EVENT_NOTE_ON)
        last_note_copy = *(const clap_event_note_t*)hdr; // safe copy
}
```

### Host callback from audio thread

**BAD:**
```c
void process(...) {
    // needs to update host about param change
    host_params->rescan(host, CLAP_PARAM_RESCAN_VALUES); // [main-thread] only!
}
```

**GOOD:**
```c
void process(...) {
    needs_param_flush.store(true, std::memory_order_release);
    host->request_callback(host); // schedules on_main_thread()
}

void on_main_thread(...) {
    if (needs_param_flush.exchange(false))
        host_params->rescan(host, CLAP_PARAM_RESCAN_VALUES); // safe here
}
```

### Wrong request_flush context

**BAD:**
```c
void process(...) {
    host_params->request_flush(host); // [thread-safe, !audio-thread]
}
```

**GOOD:**
```c
void process(const clap_process_t *process) {
    clap_event_param_value_t ev = {0};
    ev.header.size = sizeof(ev);
    ev.header.type = CLAP_EVENT_PARAM_VALUE;
    ev.header.time = 0;
    ev.param_id = PARAM_GAIN;
    ev.value = pending_gain;
    process->out_events->try_push(process->out_events, &ev.header);
}
```

### Wrong process status

**BAD:**
```c
clap_process_status process(...) {
    // always returns CONTINUE even with no active voices
    do_dsp();
    return CLAP_PROCESS_CONTINUE;
}
```

**GOOD:**
```c
clap_process_status process(...) {
    do_dsp();
    if (voice_count == 0 && tail_samples_remaining == 0)
        return CLAP_PROCESS_SLEEP;   // host may stop calling until next event
    if (tail_samples_remaining > 0)
        return CLAP_PROCESS_TAIL;
    return CLAP_PROCESS_CONTINUE;
}
```
