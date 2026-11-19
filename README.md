# OpenCode Multi-Account

Native **OpenCode V2** plugin (rewritten from the `pi-multi-account` extension). It registers one **virtual provider per account** for the same API endpoint, so several keys for one provider can be used to increase rate limits. Switch keys with `/account` or `Alt+A`, and optionally rotate automatically when a request fails.

## Load it

Install as a plugin (or place the directory under `.opencode/plugins/`):

```jsonc
{
  "plugins": [
    "opencode-multi-account",
    {
      "package": "opencode-multi-account",
      "options": { /* optional config overlay; see below */ }
    }
  ]
}
```

The package exports two parts: `.` (server) and `./tui` (terminal keybinding + picker). Loading the package loads both.

## Config

JSON is read from these paths, merged in order (later wins):

1. `~/.pi/agent/multi-account.json`
2. `~/.config/opencode/multi-account.json`
3. `<project>/.pi/multi-account.json`
4. `<project>/.opencode/multi-account.json`
5. The plugin's `options` object from `opencode.jsonc` (`options.config` is also accepted)

```json
{
  "autoSwitchOnFailure": true,
  "autoRetry": true,
  "providers": {
    "opencode": {
      "name": "OpenCode",
      "baseUrl": "https://api.open-code.ai/v1",
      "api": "openai-codex-responses",
      "sessionHeaders": { "x-opencode-session": "$sessionId", "x-client": "opencode" },
      "accounts": [
        { "label": "Primary", "apiKey": "$OPENCODE_KEY_1" },
        { "label": "Secondary", "apiKey": "$OPENCODE_KEY_2" }
      ],
      "models": [
        {
          "id": "kimi-k2.6",
          "name": "Kimi K2.6",
          "reasoning": true,
          "input": ["text", "image"],
          "cost": { "input": 1.1, "output": 4.4, "cacheRead": 0.275, "cacheWrite": 0 },
          "contextWindow": 200000,
          "maxTokens": 64000
        }
      ]
    }
  }
}
```

Provider fields: `name`, `baseUrl`, `api`, `accounts`, `models` **or** `modelUrl`, `headers`, `authHeader`, `sessionHeaders`, `compat`, and optional `package` (an explicit `@opencode/ai/providers/...` override). Account fields: `label`, `apiKey`, `headers`.

`apiKey` accepts `$VAR` / `${VAR}`, `!command` (resolved at startup in memory), or a literal. **Secrets are never written to disk by this plugin** — they are resolved into an in-memory map and passed to provider definitions through `ctx.provider.transform`. To keep keys off disk, prefer `$VAR` references; literal keys in a config file are read but never copied, cached, or logged by the plugin.

With the example above, `opencode-primary/kimi-k2.6` and `opencode-secondary/kimi-k2.6` are registered; the model ID is preserved when switching.

## Usage

| Action | How |
|---|---|
| List accounts | `/account` or `/account list` (posts the list as a synthetic status message) |
| Cycle to the next account | `/account cycle` or `Alt+A` (TUI) |
| Switch by label | `/account secondary` |
| Pick interactively | "Account: select" in the command palette (TUI) |
| Dynamic model catalog | `modelUrl` instead of `models`; fetched once, cached in memory for 30 min with stale fallback |

### Auto-switch on failure

With `autoSwitchOnFailure` (default `true`), a `retry` hook switches the session model to the next account and retries with its key. User-abort/cancel and service-level errors (502/503/504, "high demand", "overloaded", "internal server error") are left alone. With `autoRetry` (default `true`) the interrupted request is retried immediately; set it to `false` to switch keys but require the user to resend. A per-prompt budget stops after every account has been tried.

## Mapping from the pi extension

| pi concept | OpenCode V2 implementation |
|---|---|
| `pi.registerProvider` (virtual providers) | `ctx.provider.transform` → `editor.add({ info, models })`; `baseURL`/`apiKey` via `info.settings`, merged headers via `info.headers` |
| `pi.setModel` + model registry | `ctx.session.switchModel({ sessionID, model })` |
| `before_provider_headers` / `$sessionId` | `ctx.session.hook("http.request", …)` setting request headers per session |
| `message_end` + `turn_end` + `agent_before_settle` auto-switch/continue | `ctx.session.hook("retry", …)` → switch account + `event.decision = { retry: true, delay: 0 }` |
| `pi.registerCommand("account")` | `ctx.command.transform` → `editor.add({ name: "account", execute })` |
| `pi.registerShortcut("alt+a")` | TUI part: `context.keymap.layer` with `bind: "alt+a"` |
| Session state entry / status line | Session model selection persists the active account; TUI picker highlights it |
| Disk-backed model catalog cache | In-memory TTL cache + `ctx.provider.reload()` |

## Known gaps / differences

- **Credentials:** OpenCode's `ctx.integration` / `ctx.credential` store credentials on disk. To honor the "never write secrets to disk" requirement, keys are kept in memory and attached to providers via transforms instead. No `credential.activate` integration is created.
- **Status bar:** The pi footer (`acc: <label>`) is not reproduced; the TUI picker marks the active account with `●` and switching emits a toast.
- **Config hot reload:** pi reloaded config on every `session_start`; here config is read at setup and again on the 30-minute catalog refresh. Reload the plugin for immediate changes.
- **Model catalog cache:** pi persisted the `modelUrl` catalog to `multi-account-models-cache.json`; this version keeps it in memory only (no secret data either way).
- **`api` mapping:** pi API names are translated to OpenCode provider packages (`openai-completions` → `openai-compatible`, `openai-responses` → `openai`, etc.). Use the `package` field for exact control; `authHeader: false` and model-specific `compat`/`thinkingLevelMap` tuning are provider-driver concerns and are not fully reproduced.
- **Router models:** pi's `pi-virtual` router guard is implicit here — the retry hook only acts on virtual providers this plugin registered, so other plugins' router models are never rotated.
- **Project-local config trust:** OpenCode does not expose pi's project-trust signal. Config files in the project directory are read unconditionally.

## Development

```bash
bun install
bun run typecheck   # tsc --noEmit against @opencode/plugin
bun test            # pure helper tests
```

The original pi test harness drove a fake `ExtensionAPI`; it was not ported because those tests exercise pi-specific events that no longer exist. The replacement suite covers the secret-free shared logic (config merge, account mapping, selection, error classification, header expansion).

MIT © 2026 hakergeniusz
