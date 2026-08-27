# Silly Preserved Reasoning

A SillyTavern extension for preserving Kimi/Moonshot reasoning across multi-turn chats while retaining the optional partial-reasoning prefill behavior from Kimi Thinking Prefill.

## Features

### Preserved reasoning

When **Send all prior assistant reasoning back to the API** is enabled, the extension:

1. Collects real assistant chat messages from SillyTavern.
2. Excludes user, system, and `extra.isSmallSys` records.
3. Pairs them in order with outgoing `role: "assistant"` messages.
4. Re-attaches each stored `extra.reasoning` as `reasoning_content` without trimming, rewriting, or reordering the stored string.

The default preserved-reasoning model filter is:

```text
kimi,moonshot
```

Prior reasoning is billed as input tokens. SillyTavern must have stored the reasoning on the original assistant message for the extension to send it on later turns.

### Experimental GLM compatibility

Some users have reported that historical reasoning reattachment works with GLM models when `glm` is added to the **Preserved reasoning model filter**. You may try:

```text
kimi,moonshot,glm
```

This is experimental, has received very limited testing, and is not guaranteed to work. The `main` branch only re-attaches stored `reasoning_content`; it does not send GLM-specific preserved-thinking controls. A successful result may depend on defaults supplied by the selected provider or endpoint. Direct standard Z.AI normally requires `clear_thinking:false`, while proxies and managed endpoints may behave differently.

Do not add `glm` to the **Prefill model filter**. Kimi/Moonshot partial-reasoning prefill is not intended for GLM.

### Summaryception (v22) Append Only compatibility

`extra.isSmallSys` chat records are excluded from assistant-message pairing. This is important for Summaryception Append Only mode, where baked narrator records may be stored as non-user/non-system chat messages but are not real assistant replies.

The compatibility filter is intentionally generic:

```js
m && !m.is_user && !m.is_system && !m.extra?.isSmallSys
```

### Kimi/Moonshot reasoning prefill

For model IDs matching the **Prefill model filter** (default `kimi,moonshot`), the extension can:

- transform a leading `<think>...</think>` block on a trailing assistant message into `reasoning_content` with `partial: true`; or
- inject a configured partial assistant `reasoning_content` prefill on normal, regenerate, and swipe generations.

Continue generations retain the leading `<think>` transformation. Quiet and impersonate generations do not receive an injected prefill.

Tools and JSON schema requests block only the partial-prefill behavior. Historical reasoning preservation remains independent and may still run when those features are present.

## Settings

Extensions menu → **Silly Preserved Reasoning**:

- **Send all prior assistant reasoning back to the API** — enables historical reasoning preservation.
- **Preserved reasoning model filter** — comma-separated model ID substrings for historical reasoning.
- **Enable Kimi/Moonshot thinking prefill** — enables the optional transform and injection behavior.
- **reasoning_content prefill** — the partial reasoning text to inject.
- **Prefill model filter** — comma-separated model ID substrings for partial prefill.
- **Force thinking on for prefilled requests** — sets `include_reasoning` when a Kimi/Moonshot request is modified.
- **Log decisions to browser console** — enables diagnostic logging.

## Settings migration

The extension's internal settings key is `SillyPreservedReasoning`.

On first load, if existing `KimiThinkingPrefill` settings are found, they are copied non-destructively:

- existing prefill settings and toggles are retained;
- the old model filter becomes both the prefill and preserved-reasoning filters;
- the legacy settings key is left untouched so rollback remains possible.

The settings UI is loaded relative to `import.meta.url`, so the physical extension folder name does not affect `settings.html` loading.

## Recommended preserved-reasoning setup

For preserved reasoning without partial prefill:

```text
Enable Kimi/Moonshot thinking prefill: OFF
Preserved reasoning model filter: kimi,moonshot
Send all prior assistant reasoning back to the API: ON
Log decisions: OFF
```

## Verification

Temporarily enable debug logging and generate a turn. For historical reasoning you should see a browser-console message similar to:

```text
[SillyPreservedReasoning] Attached reasoning_content to 12 prior assistant message(s).
```

For partial prefill you should see:

```text
[SillyPreservedReasoning] Injected Kimi/Moonshot reasoning_content prefill: ...
```

## Provider notes

- Direct Moonshot/Kimi and compatible proxies must accept `reasoning_content` for preservation to work.
- Proxy behavior is provider-dependent; confirm that the selected provider forwards the field as expected.
- Claude is intentionally not handled because its native thinking uses structured signed blocks.

## Credits

Based on the original **Kimi Thinking Prefill** extension by Rurijian and its Kimi reasoning-prefill flow.
