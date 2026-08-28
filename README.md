# Silly Preserved Reasoning

A SillyTavern extension for preserving Kimi/Moonshot and compatible GLM reasoning across multi-turn chats while retaining the optional partial-reasoning prefill behavior from Kimi Thinking Prefill.

## Features

### Preserved reasoning

When **Send all prior assistant reasoning back to the API** is enabled, the extension:

1. Collects real assistant chat messages from SillyTavern.
2. Excludes user, system, and `extra.isSmallSys` records.
3. Pairs them in order with outgoing `role: "assistant"` messages.
4. Re-attaches each stored `extra.reasoning` as `reasoning_content` without trimming, rewriting, or reordering the stored string.

The default preserved-reasoning model filter is:

```text
kimi,moonshot,glm
```

This is the default only for genuinely new installations. Existing Silly Preserved Reasoning and legacy
Kimi Thinking Prefill settings retain their saved filters exactly. Prior reasoning is billed as input tokens.
SillyTavern must have stored the reasoning on the original assistant message for the extension to send it on later turns.

### Experimental GLM compatibility

Z.AI documents Preserved Thinking as returning complete, unmodified historical `reasoning_content` in its
original order. OpenRouter likewise accepts `reasoning_content` as an alias for historical reasoning. Limited
community testing has reported behavioral differences with GLM 5.3 Flash, but this does not confirm every
provider route, cache behavior, or refusal behavior.

The **Preserved reasoning model filter** for new installations includes:

```text
kimi,moonshot,glm
```

This support is experimental and provider-dependent. Historical reattachment and the native Z.AI
`clear_thinking:false` control are separate mechanisms. Direct standard Z.AI normally requires that control,
while OpenRouter, Coding Plan, and compatible proxies may normalize or supply preserved-thinking behavior.

For GLM models accessed through SillyTavern's **Custom** source, the optional setting
**Send `clear_thinking:false` for GLM Custom endpoints (experimental)** adds this to the actual provider request:

```json
{
  "thinking": {
    "type": "enabled",
    "clear_thinking": false
  }
}
```

The option defaults off for all users. It modifies only the per-request Custom Include Body and does not rewrite
saved Custom settings. Hapuppy forwarding and cache reporting remain unconfirmed. The option does not patch
SillyTavern and does not affect the built-in Z.AI or OpenRouter sources.

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
- **Send `clear_thinking:false` for GLM Custom endpoints (experimental)** — opt-in native Z.AI request control for matching GLM models through the Custom source.
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

The v2.1 settings migration is also non-destructive:

- existing Silly Preserved Reasoning filters and toggles remain unchanged;
- an existing settings object missing the preserved filter receives the previous safe default, `kimi,moonshot`;
- the GLM Custom request control defaults off;
- only a genuinely new installation receives `kimi,moonshot,glm` automatically.

The settings UI is loaded relative to `import.meta.url`, so the physical extension folder name does not affect `settings.html` loading.

## Recommended preserved-reasoning setup

For preserved reasoning without partial prefill:

```text
Enable Kimi/Moonshot thinking prefill: OFF
Preserved reasoning model filter: kimi,moonshot,glm
Send all prior assistant reasoning back to the API: ON
Send clear_thinking:false for GLM Custom endpoints: OFF unless explicitly testing a compatible Custom provider
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
- OpenRouter documents `reasoning_content` as an accepted alias for preserved reasoning.
- Direct standard Z.AI requires `clear_thinking:false`; this extension can add it only through the opt-in Custom-source path.
- Hapuppy and other proxy behavior is provider-dependent; confirm that the selected provider forwards the field as expected.
- Claude is intentionally not handled because its native thinking uses structured signed blocks.

## Credits

Based on the original **Kimi Thinking Prefill** extension by Rurijian and its Kimi reasoning-prefill flow.
