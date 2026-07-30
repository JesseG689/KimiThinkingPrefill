# Kimi Thinking Prefill

Client-side SillyTavern extension implementing the
[kimi-k3-jb patch](https://rentry.org/kimi-k3-jb) (`reasoning_content` thinking prefill for
Kimi/Moonshot models) without modifying any server files.

## What it does

Hooks `CHAT_COMPLETION_SETTINGS_READY` and rewrites the outgoing request payload:

1. **Injection** — when the prompt does not end on an assistant message, appends:

   ```json
   { "role": "assistant", "content": "", "reasoning_content": "<your prefill>", "partial": true }
   ```

   The model then *continues thinking* from the prefilled reasoning, per the Moonshot API's
   documented partial-prefill behavior.

2. **Preset parity (patch transform)** — if the prompt already ends on an assistant message
   whose content starts with `<think>`, the think block is moved into `reasoning_content`
   and the message is flagged `partial: true` — the exact transform the server patch adds to
   `addAssistantPrefix`. This means preset-style prefills like
   `<think>I should continue the story.` keep working unchanged.

3. **Preserved thinking (optional)** — re-attaches each prior assistant message's stored reasoning
   (`chat[i].extra.reasoning`) as `reasoning_content` in the outgoing payload, so Kimi's
   [preserved-thinking behavior](https://platform.kimi.ai/docs/guide/use-thinking-models) works in
   multi-turn chats. Off by default; requires the SillyTavern **Show thoughts** toggle to be on
   (reasoning is only persisted then). Note: prior reasoning is billed as input tokens.

## Guards (mirroring the patch)

- Only runs when the current model id matches the configurable filter (default `kimi,moonshot`,
  covers `moonshotai/kimi-k3` on OpenRouter and `kimi-k3`/`kimi-k2-*` on the direct Moonshot API).
- Skipped when JSON schema / structured output is active.
- Skipped when tools/function calling are in play.
- Injection only on normal/regenerate/swipe generations (never quiet prompts or impersonation);
  the `<think>` transform additionally applies on Continue.

## Settings

Extensions menu → **Kimi Thinking Prefill**:

- **Enable thinking prefill** — toggle for the prefill features (transform + injection).
  Independent of the re-attach toggle below; either works without the other.
- **reasoning_content prefill** — the thinking text to prefill (plain text, no `<think>` tag needed).
- **Model filter** — comma-separated substrings matched against the model id.
- **Force thinking on for prefilled requests** (default: on) — sets `include_reasoning` on requests
  this extension modifies. **Required**: with thinking disabled the model continues the seeded
  `reasoning_content` with reply text and never reasons (reply shows up inside the reasoning panel).
- **Send all prior assistant reasoning back to the API** (default: off) — the preserved-thinking
  feature above. Pairs chat messages to outgoing assistant messages 1:1 (system messages excluded).
- **Log decisions to browser console** — debug output for each guarded decision.

## Verification

Enable debug logging, generate, and check the browser console for
`[KimiThinkingPrefill] Injected reasoning_content prefill: ...`. On the server side (ST terminal
with request logging), the final message should look like:

```json
{ "role": "assistant", "content": "", "reasoning_content": "I should continue the story.", "partial": true }
```

## Notes

- Works with the direct Moonshot source, OpenRouter (Moonshot provider), and Custom endpoints —
  anywhere the model id matches the filter and the API honors `partial`/`reasoning_content`.
- The server patch is *not* required; do not run both (double transforms are harmless but pointless).
