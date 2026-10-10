# claude-opencode-go — Claude Code on OpenCode Go

A `schemaVersion: "2"`, `kind: mixin` kit for the built-in `claude` sandbox
agent that reroutes Claude Code's Anthropic API calls to an
[OpenCode Go](https://opencode.ai/docs/go/) subscription through environment
variables, with the API key resolved by the sandbox proxy.

OpenCode Go exposes an Anthropic-compatible Messages endpoint at
`https://opencode.ai/zen/go/v1/messages`, and Claude Code is one of its
[validated clients](https://opencode.ai/docs/go/) — so no translation proxy is
needed for the models served on that endpoint.

## 1. Set up your OpenCode Go API key

Subscribe to Go (or Go Plus) in the
[OpenCode console](https://opencode.ai/auth) and copy the API key. The kit
declares an `opencode-go` credential, so the real key stays on the host and
the proxy injects it as `x-api-key` for `opencode.ai` requests:

```bash
printf '%s' "$OPENCODE_API_KEY" | sbx secret set -g opencode-go
```

(or just `sbx secret set -g opencode-go` and paste the key at the prompt.)

## 2. Run it

```bash
sbx run --kit ./claude-opencode-go claude
```

From a Git repo, once pushed:

```bash
sbx run --kit "git+https://github.com/JLugagne/sbx-kits.git#dir=claude-opencode-go" claude
```

## What the mixin sets

| Variable | Value | Why |
| --- | --- | --- |
| `ANTHROPIC_BASE_URL` | `https://opencode.ai/zen/go` | Reroutes the Anthropic SDK. No trailing slash — the SDK appends `/v1/messages`, and `…/zen/go//v1/messages` answers 404. |
| `ANTHROPIC_API_KEY` | `proxy-managed` (sentinel) | Read by Claude Code and sent as `x-api-key`; the proxy swaps in the real key for `opencode.ai`. |
| `ANTHROPIC_AUTH_TOKEN` | empty | Explicitly cleared: Claude Code prefers it over `ANTHROPIC_API_KEY` when set, which would bypass the injected key. |
| `ANTHROPIC_DEFAULT_OPUS_MODEL` | `minimax-m3` | Pins every model pick to a Go model. |
| `ANTHROPIC_DEFAULT_SONNET_MODEL` | `minimax-m3` | " |
| `ANTHROPIC_DEFAULT_HAIKU_MODEL` | `minimax-m3` | " |
| `CLAUDE_CODE_SUBAGENT_MODEL` | `minimax-m3` | " |

The four model variables share a single YAML anchor (`&model`), so changing
the model is a one-line edit at the anchor in `spec.yaml`.

## Switching models

Only OpenCode Go models served on the Anthropic Messages endpoint
(`/v1/messages`) work with Claude Code directly:

| Model | Model ID | Go monthly allowance |
| --- | --- | --- |
| Claude Haiku 5.5 | `claude-haiku-5-5` | $15 |
| MiniMax M3 | `minimax-m3` | $60 |
| MiniMax M2.7 | `minimax-m2.7` | $60 |
| Qwen3.8 Max | `qwen3.8-max` | $15 |
| Qwen3.8 Flash | `qwen3.8-flash` | $30 |
| Qwen3.7 Plus | `qwen3.7-plus` | $60 |

To switch, save the kit locally, edit the anchored value:

```yaml
ANTHROPIC_DEFAULT_OPUS_MODEL: &model "qwen3.8-flash"
```

and pass `--kit` at that path. Different aliases can also use different
models (e.g. keep a cheap model on `ANTHROPIC_DEFAULT_HAIKU_MODEL` for
background tasks).

DeepSeek, GLM, Kimi and the other Go models are served on the
OpenAI-compatible `/v1/chat/completions` endpoint only. Using them with
Claude Code requires a translation proxy (Anthropic → OpenAI) running inside
the sandbox — that is a different kit design, not an env-var change.

## Notes

- The key never enters the sandbox: the container only holds the
  `proxy-managed` sentinel, and the proxy substitutes the real value on
  outbound requests to `opencode.ai`.
- An `anthropic` credential bound for other sandboxes does not interfere;
  requests to `opencode.ai` are keyed by the `opencode-go` service.
- Usage is capped per model by the Go plan; see the
  [OpenCode Go limits](https://opencode.ai/docs/go/).
