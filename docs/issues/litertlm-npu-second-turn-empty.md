<!--
Title: [NPU][MediaTek MT6989] Second message in an existing Conversation returns empty output (onDone ~20 ms after decode)
Labels: bug, npu
-->

### Summary

On a MediaTek Dimensity 9300+ (MT6989) with the official MT6989 NPU bundle
`Gemma3-1B-IT_q4_ekv1280_mt6989.litertlm`, the first message in a `Conversation`
works, but the **second** message returns an empty response. `onDone()` fires
about 20 ms after `RunDecodeAsync`, with no tokens and no error.

Creating a **new** `Conversation` with the same history (`initialMessages`) and
sending the same message works. So the engine, the NPU dispatch and the bundle
are usable — it is the second turn of a **live** conversation that produces no
output.

### Environment

- Device: Xiaomi 14T Pro (rothko), SoC `MT6989` (APU 790), Android 16, SDK 36,
  4 KB pages
- Bundle: `Gemma3-1B-IT_q4_ekv1280_mt6989.litertlm` (1,032,830,976 bytes),
  from `litert-community/Gemma3-1B-IT`
- Runtime A: `litertlm-android` 0.10.0 (Box 3.5.5)
- Runtime B: `litertlm-android` 0.18.0 (Edge Gallery 1.0.20) — same symptom,
  reported by the user, no logs captured yet
- Backend: NPU, Neuron SDK 8.2.26

Engine settings as logged by the runtime:

```text
MainExecutorSettings: backend: NPU
max_tokens: 1024
kv_increment_size: 16
prefill_chunk_size: -1
clear_kv_cache_before_prefill: true
```

### Steps to reproduce

1. Create an `Engine` with `Backend.NPU(...)` on MT6989 and the bundle above.
2. `engine.createConversation(...)`.
3. `conversation.sendMessageAsync("Reply with exactly one word: OK", callback)` —
   a reply is produced.
4. Send the same message again on the same `Conversation`.
5. `onMessage` is never called; `onDone` fires after ~20 ms.

### Observed logcat (runtime A)

```text
18:45:46.999 18055 I litert : [environment.cc:30] Creating LiteRT environment with options
18:45:46.999 18055 I litert : [npu_registry.cc:30] NPU accelerator registered.
18:45:47.007 18055 I litert : [litert_dispatch.cc:149] Loading shared library: .../npu_dispatch/libLiteRtDispatch_MediaTek.so
18:45:47.015 18055 I litert : [neuron_adapter_api.cc:331] Magic number: -1
18:45:47.403 18055 I litert : [neuron_adapter_api.cc:272] Neuron api version: 8.2.26
18:45:47.405 18055 I litert : [schema_resolver.h:112] There are 2 subgraphs in the bytecode
18:45:49.336 18055 W litert : [npu_registry.cc:34] NPU accelerator could not be loaded and registered: kLiteRtStatusErrorInvalidArgument.

# turn 1
18:46:13.613 19068 I native: session_basic.cc:395] RunPrefillAsync status: OK
18:46:13.613 19068 I native: session_basic.cc:515] RunDecodeAsync
# tokens are produced

# turn 2, same Conversation
18:46:33.667 19068 I native: session_basic.cc:395] RunPrefillAsync status: OK
18:46:33.667 19068 I native: session_basic.cc:515] RunDecodeAsync
18:46:33.687 19068 W ...   onDone() with an empty message (≈20 ms later)
```

No `Total decode latency` / `Decode num tokens` line is emitted for either turn,
i.e. the session is never closed between turns.

### Controls

| Scenario | Result |
|---|---|
| NPU, 2nd message, same `Conversation` | **empty**, `onDone` after ~20 ms |
| NPU, new `Conversation` with `initialMessages` = history (app restarted, chat restored) | **works** |
| NPU, new `Conversation`, empty history ("New chat") | **works** |
| CPU, generic `gemma3-1b-it-int4.litertlm`, 2nd message, same `Conversation` | **works** |

Note that `npu_registry.cc:34` reports that the NPU accelerator could not be
registered, yet the dispatch path is used and turn 1 runs on the NPU, so this
warning looks unrelated to the symptom.

### Questions

1. Is multi-turn use of a single `Conversation` supported on NPU with a **static**
   KV-cache bundle (`ekv1280`), or is a conversation effectively single-turn and
   apps are expected to recreate it per turn?
2. If multi-turn is supported, could this be related to the KV-cache / attention
   mask handling on the NPU path? Issue #4184 (no clamp of `max_num_tokens` to the
   compiled KV cache length) and #3508 (NPU prefill chunks) look adjacent, but
   neither matches exactly: here `max_tokens` is 1024 < 1280 and turn 1 is correct.
3. Is there any intention to publish an MT6989 bundle with a **dynamic** KV cache
   (v0.18.0 added on-demand KV growth on NPU)? There is currently only the single
   static `Gemma3-1B-IT_q4_ekv1280_mt6989` bundle for MediaTek on
   `litert-community`, while Qualcomm/Intel/Google Tensor have several.
4. As a workaround, is "close the existing conversation, create a new one with
   `initialMessages`, then send" the recommended pattern on NPU, or does it risk
   the state-loss behaviour described in #2807?

### Additional context

Happy to run further tests on the device (including a locally built runtime) and
provide full logcat or a minimal repro app if useful.
