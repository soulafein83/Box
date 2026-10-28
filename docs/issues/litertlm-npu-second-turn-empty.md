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
- Runtime A: `litertlm-android` 0.10.0 (third-party app, logcat attached below)
- Runtime B: `litertlm-android` 0.18.1 (Edge Gallery 1.0.22, built from source with
  the MediaTek dispatch runtime) — **same symptom, see "Results on 0.18.1"**
- Backend: NPU (MediaTek NeuroPilot), Neuron SDK 8.2.26
- `MT6989` is listed as a supported SoC in the LiteRT MediaTek guide
  (<https://developers.google.com/edge/litert/next/mediatek>), and LiteRT-LM
  documents MediaTek NeuroPilot as a supported NPU vendor
  (<https://developers.google.com/edge/litert/next/litert_lm_npu>), so this is a
  documented, supported combination.

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

### Reproducing on Edge Gallery (no shipped MediaTek NPU runtime)

`google-ai-edge/gallery` publishes vendor-specific APKs for Qualcomm
(`sm8550`, `sm8650`, `sm8750`, `sm8850`) and Google Tensor (`tensor-g5`,
`tensor-g6`). Those contain the vendor dispatch library, for example:

```text
# ai-edge-gallery-sm8750.apk (1.0.20)
lib/arm64-v8a/libLiteRtDispatch_Qualcomm.so
```

The generic `ai-edge-gallery.apk` and the source tree contain no dispatch
library at all, and there is **no MediaTek variant**. So to run the MT6989 NPU
on Edge Gallery you have to build from source and add the MediaTek runtime:

```kotlin
// Android/src/app/build.gradle.kts
implementation(libs.litertlm)
implementation("com.google.ai.edge.litert:litert-npu-runtime-mediatek:2.3.2")
```

which produces:

```text
lib/arm64-v8a/libLiteRtCompilerPlugin_MediaTek.so
lib/arm64-v8a/libLiteRtDispatch_MediaTek.so
```

`LlmChatModelHelper` passes `Backend.NPU(nativeLibraryDir = ...)`, so the AAR
alone is enough. This is what I use for the runtime B measurements below.

### Results on 0.18.1 (Edge Gallery 1.0.22)

Same device, same bundle, fresh install, NPU selected at import time
(`compatible_accelerators: "NPU"`). Model load succeeds and the dispatch path
runs entirely on the NPU:

```text
20:26:57.364 AGLlmChatModelHelper: Preferred backend: NPU(nativeLibraryDir=/data/app/.../com.google.aiedge.gallery.npudbg-.../lib/arm64)
20:27:00.966 litert: [litert_dispatch_invocation_context.cc:266] Neuron SDK version check passed: bytecode 8.2.26, runtime 8.2.26
20:27:00.966 litert: [schema_resolver.h:112] There are 26 subgraphs in the bytecode
20:27:01.260 AGModelManagerViewModel: Model 'Gemma3-1B-IT_q4_ekv1280_mt6989-1.litertlm' initialized successfully
```

Two turns, sent from the chat screen. The latency shown under each answer is the
app's own end-to-end measurement for that message:

| Turn | Prompt | Answer | App-reported latency |
|---|---|---|---|
| 1 | `Reply with exactly on Word: OK` | `OK` | 135.0 ms |
| 2 | `Reply with exactly on Word: OK` | *(empty)* | 91.0 ms |

No exception and no error dialog — `onDone` simply arrives with no tokens, so the
second bubble stays empty. `user_data.pb` of the session stores both user
messages and no model message for turn 2.

Note: 0.18.1 does not emit the `session_basic.cc: RunPrefillAsync` /
`RunDecodeAsync` lines at default verbosity (0.10.0 did), so the evidence here is
the empty message plus the app latency, not a native decode log.

Also, 0.18.1 no longer has the "only one session is supported at a time"
restriction, so an app can close the old conversation and create a new one. That
makes a per-turn rebuild workable as a workaround, but the underlying second-turn
failure is unchanged in this version.

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
