# JUCE Violation Catalogue

Full reference for `juce-review` skill. Each section shows the BAD pattern, why it is wrong, and the GOOD replacement.

---

## 1. String and DBG Violations

### juce::String in processBlock

`juce::String` operations can allocate heap memory and `DBG()` also routes through JUCE
logging. Any construction, concatenation, conversion, or logging in the audio thread can block.

**BAD**
```cpp
void processBlock(AudioBuffer<float>& buffer, MidiBuffer& midi) override
{
    juce::String msg = "Level: " + juce::String(buffer.getMagnitude(0, 0, buffer.getNumSamples()));
    logger.writeToLog(msg);  // double violation: String + file I/O
}
```

**GOOD**
```cpp
// In prepareToPlay or editor, format strings there.
// Pass a flag or scalar to the audio thread via std::atomic.
std::atomic<float> lastMagnitude { 0.0f };

void processBlock(AudioBuffer<float>& buffer, MidiBuffer& midi) override
{
    lastMagnitude.store(buffer.getMagnitude(0, 0, buffer.getNumSamples()),
                        std::memory_order_relaxed);
}

// Editor timer callback reads lastMagnitude and formats the string there.
```

### DBG() Macro

`DBG()` expands to `juce::Logger::writeToLog(juce::String(...))` in debug builds — both String allocation and file I/O.

**BAD**
```cpp
void processBlock(...) override {
    DBG("processBlock called, samples=" << buffer.getNumSamples());
}
```

**GOOD**
Remove `DBG()` entirely from audio-thread code. If you need diagnostics, write a scalar into an atomic and read it from the editor or a timer.

---

## 2. Parameter Access Patterns

### Direct APVTS Access from Audio Thread

`AudioProcessorValueTreeState` uses `ValueTree` internally and is not the realtime data path.
Keep `apvts.state`, listeners, attachments, and parameter lookup out of `processBlock`; cache raw
parameter pointers and read those atomics in the callback.

**BAD**
```cpp
void processBlock(AudioBuffer<float>& buffer, MidiBuffer&) override
{
    auto* param = apvts.getParameter("gain");   // non-realtime lookup path
    float gain = param->getValue();
}
```

**GOOD — cache raw pointers once after APVTS construction**
```cpp
// In the processor header:
std::atomic<float>* gainParam = nullptr;

// In the processor constructor, after apvts has been constructed:
MyProcessor::MyProcessor()
{
    gainParam = apvts.getRawParameterValue("gain");
}

// In processBlock:
void processBlock(AudioBuffer<float>& buffer, MidiBuffer&) override
{
    float gain = gainParam->load(std::memory_order_relaxed);  // lock-free read
    // ... apply gain
}
```

`getRawParameterValue` returns a pointer to the `std::atomic<float>` used for the parameter's
raw value. Reading it with `memory_order_relaxed` is allocation-free.

---

## 3. Thread-Safe Component Updates

### MessageManager from Audio Thread

Message-thread calls, locks, and synchronous dispatch are not realtime-safe. `callAsync()` is
thread-safe but allocates a message, so it still does not belong in `processBlock`.

**BAD**
```cpp
void processBlock(...) override
{
    juce::MessageManager::getInstance()->callFunctionOnMessageThread(...);  // deadlock risk
}
```

**GOOD outside the audio callback — use callAsync**
```cpp
// Post a lambda to the message thread from a non-audio thread:
juce::MessageManager::callAsync([this] {
    editor->updateMeter(currentLevel.load());
});
```

Note: `callAsync` itself allocates (it creates a heap-allocated message object). Do **not** call it from inside `processBlock`. Instead, post from a background monitoring thread or from a timer that polls an atomic.

### MessageManagerLock Caveat

`MessageManagerLock` acquired on the audio thread can cause priority inversion (message thread holds the MM lock; audio thread waits). Never use it in `processBlock`.

### Recommended: AbstractFifo for Audio→UI Communication

```cpp
// In processor header:
juce::AbstractFifo fifo { 1024 };
std::array<float, 1024> fifoBuffer;

// Audio thread — writes level data:
void processBlock(AudioBuffer<float>& buffer, MidiBuffer&) override
{
    int start1, size1, start2, size2;
    fifo.prepareToWrite(1, start1, size1, start2, size2);
    if (size1 > 0) {
        fifoBuffer[start1] = currentLevel;
        fifo.finishedWrite(1);
    }
}

// Editor timer (message thread) — reads and updates UI:
void timerCallback() override
{
    int start1, size1, start2, size2;
    fifo.prepareToRead(fifo.getNumReady(), start1, size1, start2, size2);
    for (int i = 0; i < size1; ++i) consume(fifoBuffer[start1 + i]);
    for (int i = 0; i < size2; ++i) consume(fifoBuffer[start2 + i]);
    fifo.finishedRead(size1 + size2);
    repaint();
}
```

---

## 4. ValueTree and Listeners

### ValueTree Listener Thread Assumption

`ValueTree::Listener` callbacks are dispatched on the thread that modifies the tree. If your code modifies the tree from the message thread (which is the correct approach), your listener runs on the message thread — which is fine. But if you mistakenly call `apvts.state.setProperty(...)` from the audio thread, listeners fire on the audio thread.

**BAD**
```cpp
void processBlock(...) override
{
    apvts.state.setProperty("playhead", position, nullptr);  // fires listeners on audio thread
}
```

**GOOD**
Keep all `ValueTree` modifications on the message thread. Pass data to/from audio thread via atomics or lock-free queues only.

---

## 5. MIDI Buffer Iteration Safety

### Modifying MidiBuffer While Iterating

**BAD**
```cpp
void processBlock(AudioBuffer<float>& buffer, MidiBuffer& midi) override
{
    for (auto meta : midi)
    {
        if (meta.getMessage().isNoteOn())
            midi.addEvent(MidiMessage::noteOff(1, meta.getMessage().getNoteNumber()), meta.samplePosition);
        // ^ iterator invalidated after addEvent
    }
}
```

**GOOD — use a preallocated member buffer**
```cpp
// Member: juce::MidiBuffer processedMidi;

void processBlock(AudioBuffer<float>& buffer, MidiBuffer& midi) override
{
    processedMidi.clear();
    for (auto meta : midi)
    {
        auto msg = meta.getMessage();
        if (msg.isNoteOn())
            processedMidi.addEvent(juce::MidiMessage::noteOff(1, msg.getNoteNumber()), meta.samplePosition);
        else
            processedMidi.addEvent(msg, meta.samplePosition);
    }
    midi.swapWith(processedMidi);
}
```

`MidiBuffer::addEvent()` can grow internal storage. If the event count can exceed prior
capacity, use a bounded fixed-size event array or pre-warm the member `MidiBuffer` outside
the audio callback.

---

## 6. AudioBuffer Management

### setSize in processBlock

`AudioBuffer::setSize` calls `malloc`/`free` when the requested size exceeds current capacity.

**BAD**
```cpp
void processBlock(AudioBuffer<float>& buffer, MidiBuffer& midi) override
{
    scratchBuffer.setSize(buffer.getNumChannels(), buffer.getNumSamples());  // may allocate
}
```

**GOOD**
```cpp
void prepareToPlay(double sampleRate, int samplesPerBlock) override
{
    scratchBuffer.setSize(getTotalNumOutputChannels(), samplesPerBlock);  // safe here
    scratchBuffer.clear();
}

void processBlock(AudioBuffer<float>& buffer, MidiBuffer& midi) override
{
    // scratchBuffer is already the right size — use directly
}
```

---

## 7. prepareToPlay Reset Completeness

Double calls to `prepareToPlay` happen legitimately (sample rate change, buffer size change, DAW transport reset). All stateful DSP objects must be fully reset.

**BAD — filter state not reset**
```cpp
void prepareToPlay(double sampleRate, int samplesPerBlock) override
{
    filter.setCoefficients(...);  // sets coefficients but leaves filter history intact
}
```

**GOOD**
```cpp
void prepareToPlay(double sampleRate, int samplesPerBlock) override
{
    juce::dsp::ProcessSpec spec { sampleRate, (uint32) samplesPerBlock, (uint32) getTotalNumOutputChannels() };
    filter.prepare(spec);   // prepare resets state
    filter.reset();         // belt-and-suspenders for JUCE DSP module filters
    envelope.reset();
    delayLine.reset();
    playHeadPosition = 0;
}
```

Checklist for `prepareToPlay`:
- All `juce::dsp::*` processors: call `prepare(spec)` then `reset()`
- Envelope generators: reset phase and state
- Delay lines: clear buffer and reset write pointer
- Oscillator phases: reset to 0
- Smoothed values: call `reset(sampleRate, rampLengthSeconds)`
- Scratch buffers: call `setSize` with worst-case block size

---

## 8. Recommended JUCE Patterns Summary

| Concern | Recommended Pattern |
|---------|---------------------|
| Parameter reads in processBlock | `getRawParameterValue` cached once after APVTS construction; `atomic::load(relaxed)` |
| UI meter / level display | `std::atomic<float>` written by audio thread; editor timer reads it |
| Audio→UI event stream | `juce::AbstractFifo` + fixed array; editor timer drains it |
| UI→audio command (e.g. load preset) | `juce::dsp::DelayLine`-style atomic flag or SPSC queue; audio thread polls |
| Async message thread work | `juce::MessageManager::callAsync` from non-audio threads only |
| MIDI processing | Copy-then-swap pattern; never mutate while iterating |
| DSP state initialisation | Always in `prepareToPlay`; always call `reset()` |
