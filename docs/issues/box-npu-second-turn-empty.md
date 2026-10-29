<!--
Title: NPU (MT6989): second message in the same chat returns an empty reply; the "NPU window slide" rebuild fails with FAILED_PRECONDITION
Labels: bug
-->

> **Published:** https://github.com/jegly/Box/issues/190

**Describe the bug:**

On a MediaTek Dimensity 9300+ (MT6989) with the official MT6989 NPU bundle
`Gemma3-1B-IT_q4_ekv1280_mt6989.litertlm`, the **first** message in a new chat
works, but the **second** message in the same chat returns an empty reply. The
reply appears almost instantly (the UI still shows a latency in ms).

Logcat shows the app trying to rebuild the conversation and failing:

```text
18:46:33.615 18055 W AGLlmChatModelHelper: NPU window slide failed; continuing without it
18:46:33.615 18055 W AGLlmChatModelHelper: LiteRtLmJniException: Failed to create conversation:
              FAILED_PRECONDITION: A session already exists. Only one session is supported at a time.
              Please delete the existing ...
                at LiteRtLmJni.nativeCreateConversation(Native Method)
18:46:33.667 19068 I native: session_basic.cc:395] RunPrefillAsync status: OK
18:46:33.667 19068 I native: session_basic.cc:515] RunDecodeAsync
18:46:33.687 19068 W AGLlmChatModelHelper: empty-recovery rebuild failed; continuing
18:46:33.687 19068 W AGLlmChatModelHelper: Failed to create conversation: FAILED_PRECONDITION: A session already exists...
                at Conversation$JniMessageCallbackImpl.onDone(...)
```

**To Reproduce:**

1. Install Box 3.5.5 (`versionCode=49`), import `Gemma3-1B-IT_q4_ekv1280_mt6989.litertlm`.
2. Select the NPU accelerator and start a new chat.
3. Send `Reply with exactly one word: OK` — the model answers.
4. Send the same message again.
5. The second answer is empty; no error is shown.

**Expected behavior:**

The second message produces a normal reply, exactly like the first one.

**Device & App Information:**

- Device: Xiaomi 14T Pro (rothko), SoC MT6989, APU 790
- Android: 16 (HyperOS 3.0.302.0.WNNEUXM), SDK 36
- App: Box 3.5.5, `versionCode=49`, `targetSdk=37`
- Model: `Gemma3-1B-IT_q4_ekv1280_mt6989-1.litertlm` (`litert-community/Gemma3-1B-IT`)
- Backend: NPU (`MainExecutorSettings: backend: NPU`), Neuron SDK 8.2.26

**Additional context:**

### What works and what does not (measured on the same device)

| Scenario | Result |
|---|---|
| NPU, 2nd message in the same chat | **empty reply** (onDone ~20 ms after `RunDecodeAsync`) |
| NPU, force-stop the app, reopen, restore the same chat, send next message | **works** |
| NPU, "New chat" (fresh conversation, empty history) | **works** |
| CPU, generic `gemma3-1b-it-int4.litertlm`, 2nd message in the same chat | **works** |

Engine settings captured at load time:

```text
MainExecutorSettings: backend: NPU
max_tokens: 1024
kv_increment_size: 16
clear_kv_cache_before_prefill: true
model_path: .../__imports/Gemma3-1B-IT_q4_ekv1280_mt6989-1.litertlm
```

### Two separate problems

1. **The second turn of a live conversation produces no tokens on NPU.** This is
   not specific to Box: it reproduces in the upstream Edge Gallery app
   (1.0.22, `litertlm` 0.18.1) with the same bundle, where turn 1 answers `OK` in
   135 ms and turn 2 returns an empty reply in 91 ms. A conversation that is
   *created fresh* with the same history (`initialMessages`) answers correctly,
   so the engine and the NPU bundle are fine; only the second turn of a live
   conversation fails. This points at LiteRT-LM rather than at Box, so it is
   being reported there separately.

2. **The workaround implemented in Box cannot work as written.** The "NPU window
   slide" path calls `engine.createConversation(...)` while the previous
   conversation is still alive. LiteRT-LM allows one session per engine, so the
   call throws `FAILED_PRECONDITION`. The exception is swallowed
   ("continuing without it") and the message is sent to the conversation that
   can no longer produce output. The "empty-recovery" path repeats exactly the
   same mistake, so it can never recover.

### Suggested fix for (2)

Rebuild the conversation for every NPU turn instead of trying to slide it, and
close the previous one first:

```kotlin
val old = instance.conversation
runCatching { old.cancelProcess() }
if (old.isAlive) runCatching { old.close() }   // close() throws if already closed

val conversation = try {
  createConversationWithRetry(
    engine,
    baseConfig.copy(initialMessages = historyWithoutCurrentMessage),
  )
} catch (e: Exception) {
  onError("Could not rebuild the NPU context")   // do not send into a dead session
  return
}
instance.conversation = conversation
```

Notes that matter:

- `close()` must be called **before** `createConversation`; wrap it in
  `runCatching` because it throws `IllegalStateException` when the conversation
  is already closed.
- Retry `createConversation` a few times with a short delay — NPU session
  teardown is not always immediate.
- Do the rebuild on `Dispatchers.IO`, **not** from `onDone`: closing a native
  conversation from the message-callback thread can self-join the LiteRT-LM
  worker. (Upstream 1.0.22 offloads `close()` to an IO dispatcher for this
  reason.)
- Trim `initialMessages` so the history fits the bundle's static KV window
  (1280 tokens here); otherwise re-prefill will eventually exceed it.
- Upgrading `litertlm` to 0.18.x also removes the "only one session" restriction,
  which makes the close/create sequence race-free.

### Workarounds for users right now

- Use the CPU backend for multi-turn chats (works, but slower).
- Restart the app (or start a new chat) before each message on NPU.
