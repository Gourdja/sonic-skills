# Web Audio Violation Patterns and Safe Alternatives

Reference for the `webaudio-review` skill. Examples are JavaScript/TypeScript targeting the
Web Audio API and the AudioWorklet system.

## Table of Contents
1. [ScriptProcessorNode (deprecated)](#1-scriptprocessornode-deprecated)
2. [Allocations inside process()](#2-allocations-inside-process)
3. [postMessage overhead](#3-postmessage-overhead)
4. [SharedArrayBuffer + integer Atomics](#4-sharedarraybuffer--integer-atomics)
5. [DOM / window access from worklet](#5-dom--window-access-from-worklet)
6. [Autoplay policy](#6-autoplay-policy)
7. [AudioNode graph changes from worklet](#7-audionode-graph-changes-from-worklet)
8. [AudioWorklet module registration](#8-audioworklet-module-registration)

---

## 1. ScriptProcessorNode (deprecated)

`ScriptProcessorNode` runs its `onaudioprocess` callback on the **main thread**. Any UI work,
layout, or garbage collection on the main thread competes directly with your audio callback.

```js
// BAD — deprecated, runs on main thread
const processor = audioContext.createScriptProcessor(256, 1, 1);
processor.onaudioprocess = (e) => {
  const input  = e.inputBuffer.getChannelData(0);
  const output = e.outputBuffer.getChannelData(0);
  for (let i = 0; i < input.length; i++) {
    output[i] = input[i] * 0.5;
  }
};
source.connect(processor).connect(audioContext.destination);
```

```js
// GOOD — AudioWorkletProcessor runs on a dedicated worklet thread
// worklet-processor.js
class GainProcessor extends AudioWorkletProcessor {
  process(inputs, outputs) {
    const input  = inputs[0][0];
    const output = outputs[0][0];
    if (input) {
      for (let i = 0; i < output.length; i++) {
        output[i] = input[i] * 0.5;
      }
    }
    return true; // keep processor alive
  }
}
registerProcessor('gain-processor', GainProcessor);
```

---

## 2. Allocations inside process()

Every `new Float32Array(...)`, object literal `{}`, array `[]`, or closure created inside
`process()` is a GC-eligible allocation. Garbage collection and object churn can add
unpredictable work to the render thread.

```js
// BAD — allocates a new Float32Array every render quantum
class BadProcessor extends AudioWorkletProcessor {
  process(inputs, outputs) {
    const tmp = new Float32Array(outputs[0]?.[0]?.length ?? 128); // GC allocation every block
    // ...
    return true;
  }
}
```

```js
// GOOD — pre-allocate in constructor, reuse across calls
class GoodProcessor extends AudioWorkletProcessor {
  constructor(options) {
    super(options);
    const maxFrames = options.processorOptions?.maxFramesPerQuantum ?? 128;
    this._tmp = new Float32Array(maxFrames); // allocate once
  }

  process(inputs, outputs) {
    // reuse this._tmp — no allocation
    return true;
  }
}
```

---

## 3. postMessage overhead

Calling `this.port.postMessage()` on every render quantum (currently every ~2.9ms at
44.1 kHz with 128-frame quanta) creates a stream of structured-clone messages that crosses the thread
boundary and can accumulate.

```js
// BAD — postMessage on every single render quantum
class BadMeter extends AudioWorkletProcessor {
  process(inputs, outputs) {
    const rms = computeRms(inputs[0][0]);
    this.port.postMessage({ rms }); // called ~344 times/sec
    return true;
  }
}
```

```js
// GOOD — batch at a lower rate (e.g. every 50 blocks ≈ 6 Hz)
class GoodMeter extends AudioWorkletProcessor {
  constructor(options) {
    super(options);
    this._blockCount = 0;
    this._rmsAccum   = 0;
  }

  process(inputs, outputs) {
    this._rmsAccum += computeRms(inputs[0][0]);
    this._blockCount++;
    if (this._blockCount >= 50) {
      this.port.postMessage({ rms: this._rmsAccum / 50 });
      this._blockCount = 0;
      this._rmsAccum   = 0;
    }
    return true;
  }
}
```

---

## 4. SharedArrayBuffer + integer Atomics

For low-latency parameter control without postMessage overhead, use `SharedArrayBuffer` with
an integer atomic protocol for safe reads and writes across threads. `Atomics.load()` and
`Atomics.store()` operate on integer typed arrays; they do not accept `Float32Array`. Requires
the page to be cross-origin isolated (`COOP`/`COEP` headers).

```js
// BAD — raw SharedArrayBuffer read without Atomics: race-prone shared state
class BadParamReader extends AudioWorkletProcessor {
  constructor(options) {
    super(options);
    this._buf = options.processorOptions.sharedBuffer; // Float32Array view
  }

  process(inputs, outputs) {
    const gain = this._buf[0]; // not synchronized with the writer
    // ...
    return true;
  }
}
```

```js
// GOOD — store a scaled integer parameter with Atomics

// --- main thread setup ---
const sharedBuffer = new SharedArrayBuffer(4);
const controlView  = new Int32Array(sharedBuffer);

await audioContext.audioWorklet.addModule('gain-processor.js');
const node = new AudioWorkletNode(audioContext, 'gain-processor', {
  processorOptions: { sharedBuffer },
});

// Write gain from main thread (e.g. on slider input)
function setGain(value) {
  const scaled = Math.round(Math.max(0, Math.min(value, 4)) * 1_000_000);
  Atomics.store(controlView, 0, scaled);
}

// --- worklet side (gain-processor.js) ---
class GainProcessor extends AudioWorkletProcessor {
  constructor(options) {
    super(options);
    this._controlView = new Int32Array(options.processorOptions.sharedBuffer);
  }

  process(inputs, outputs) {
    const gain = Atomics.load(this._controlView, 0) / 1_000_000;

    const input  = inputs[0][0];
    const output = outputs[0][0];
    if (input) {
      for (let i = 0; i < output.length; i++) {
        output[i] = input[i] * gain;
      }
    }
    return true;
  }
}
registerProcessor('gain-processor', GainProcessor);
```

If exact float bit patterns are required, atomically store the bits in a `Uint32Array` and
reinterpret them through a private non-shared `ArrayBuffer` on the reading side. Do not write
a `Float32Array` view and assume a later integer atomic read makes the float write atomic.

---

## 5. DOM / window access from worklet

The AudioWorklet global scope is not `Window`. It has no `document`, no `window`, and no DOM
API. Do resource loading on the main thread or in a worker, then pass decoded data or a
`SharedArrayBuffer` to the processor.

```js
// BAD — throws ReferenceError in worklet scope
class BadProcessor extends AudioWorkletProcessor {
  process(inputs, outputs) {
    console.log(window.location.href); // ReferenceError: window is not defined
    document.title = 'processing';     // ReferenceError: document is not defined
    return true;
  }
}
```

```js
// GOOD — communicate back to main thread via port for any DOM needs
class GoodProcessor extends AudioWorkletProcessor {
  process(inputs, outputs) {
    if (someCondition) {
      this.port.postMessage({ type: 'update-title', text: 'processing' });
    }
    return true;
  }
}

// In main thread:
node.port.onmessage = (e) => {
  if (e.data.type === 'update-title') {
    document.title = e.data.text; // DOM access only on main thread
  }
};
```

---

## 6. Autoplay policy

Browsers suspend `AudioContext` by default until a user gesture is detected. Code that creates
a context and starts playback without waiting for user interaction will produce no audio.

```js
// BAD — context may be suspended; audio never plays
const audioContext = new AudioContext();
source.start(); // context is suspended — silent
```

```js
// GOOD — resume on first user gesture
const audioContext = new AudioContext();

document.addEventListener('click', async () => {
  if (audioContext.state === 'suspended') {
    await audioContext.resume();
  }
}, { once: true });

// Or: gate playback on context state
async function startAudio() {
  if (audioContext.state !== 'running') {
    await audioContext.resume();
  }
  source.start();
}
```

---

## 7. AudioNode graph changes from worklet

`AudioNode.connect()` and `AudioNode.disconnect()` are main-thread operations and must not be
called from inside the worklet. Attempting to do so will throw or silently fail depending on
the browser.

```js
// BAD — graph mutation from inside process() — not valid
class BadRouter extends AudioWorkletProcessor {
  process(inputs, outputs) {
    if (someCondition) {
      this.outputNode.connect(this.busA); // undefined + main-thread-only
    }
    return true;
  }
}
```

```js
// GOOD — send a message to main thread; apply graph change there
// --- worklet ---
class GoodRouter extends AudioWorkletProcessor {
  process(inputs, outputs) {
    if (someCondition) {
      this.port.postMessage({ type: 'route', target: 'busA' });
    }
    return true;
  }
}

// --- main thread ---
node.port.onmessage = (e) => {
  if (e.data.type === 'route') {
    node.disconnect();
    node.connect(e.data.target === 'busA' ? busA : busB);
  }
};
```

---

## 8. AudioWorklet module registration

The worklet module must be registered with `audioContext.audioWorklet.addModule()` before
creating an `AudioWorkletNode`. The file runs in the worklet scope, not the main scope.

```js
// BAD — creating AudioWorkletNode before addModule resolves
const node = new AudioWorkletNode(audioContext, 'my-processor'); // throws InvalidStateError
await audioContext.audioWorklet.addModule('processor.js');
```

```js
// GOOD — always await addModule before constructing the node
await audioContext.audioWorklet.addModule('processor.js');

const node = new AudioWorkletNode(audioContext, 'my-processor', {
  numberOfInputs:  1,
  numberOfOutputs: 1,
  outputChannelCount: [2],
});

node.connect(audioContext.destination);

// processor.js (separate file — loaded into worklet scope)
class MyProcessor extends AudioWorkletProcessor {
  static get parameterDescriptors() {
    return [{ name: 'gain', defaultValue: 1, minValue: 0, maxValue: 2 }];
  }

  process(inputs, outputs, parameters) {
    const gainValues = parameters.gain;
    const input  = inputs[0][0];
    const output = outputs[0][0];
    if (input) {
      for (let i = 0; i < output.length; i++) {
        const gain = gainValues.length === 1 ? gainValues[0] : gainValues[i];
        output[i] = input[i] * gain;
      }
    }
    return true;
  }
}
registerProcessor('my-processor', MyProcessor);
```
