# dsh — DeepSeek Harness sandbox kit

A `schemaVersion: "2"`, `kind: sandbox` kit for
[DeepSeek Harness](https://github.com/deepseek-ai/deepseek-harness) (`dsh`),
modeled on the `claude/` kit in
[docker/sbx-kits-contrib](https://github.com/docker/sbx-kits-contrib/blob/main/claude/):
a base image built from the `Dockerfile` in this directory, referenced from
`spec.yaml`, plus declarative network permissions and credential injection.

`dsh` isn't one of sbx's built-in agents, so — unlike a `kind: mixin` that
extends an existing agent — this kit defines a full sandbox agent from
scratch: its own image, entrypoint, and network policy.

## 1. Build and publish the image

`sandbox.image` in `spec.yaml` must point at an already-published tag —
`sbx` doesn't build the Dockerfile itself.

```bash
docker build -t ghcr.io/jlugagne/dsh-sbx-kit:latest --push .
```

Then edit `spec.yaml` and replace `ghcr.io/jlugagne/dsh-sbx-kit:latest` with
your actual tag.

## 2. Set up your DeepSeek API key

The kit declares a `deepseek` credential (see below), so the real key
stays on the host and the proxy injects it as a Bearer token for
`api.deepseek.com` requests:

```bash
echo "$DEEPSEEK_API_KEY" | sbx secret set -g deepseek
```

## 3. Run it

```bash
sbx run --kit ./dsh dsh
```

(`dsh` is both the kit's name and the agent positional here — for a
`kind: sandbox` kit, the agent argument is the kit's own name.)

From a Git repo, once pushed:

```bash
sbx run --kit "git+https://github.com/JLugagne/sbx-kits.git#dir=dsh" dsh
```

The entrypoint runs `dsh web --no-open` by default. Interactive attach
(`command.interactive: []`) runs `dsh web` with no extra args, which opens a
browser from inside the sandbox — harmless but unnecessary there, so
`--no-open` on the default path is the one that matters for headless/detached
runs.

## Reaching the Web UI from the host

`dsh web` only binds `127.0.0.1` (it refuses `0.0.0.0` on purpose), while the
sandbox runtime forwards a published host port to the sandbox's external
addresses. The kit therefore ships `files/home/.dsh/sbx-relay.js` and starts
it with `setup.startup`: it listens on the sandbox interfaces and pipes every
connection to the loopback web server. Without it, the published port accepts
the TCP connection and then resets it (the runtime's dial to the sandbox
address is refused).

The host port is ephemeral, so read it from `sbx ls` / `sbx ports <name>`
(or pin it explicitly, e.g. `sbx ports <name> --publish 127.0.0.1:32772:3080`).
The first visit needs the tokenized URL that `dsh web` prints at startup
(`dsh web: http://127.0.0.1:3080/?token=…`); the token mints a signed cookie,
and any later visit to the same `host:port` authority works without it.

## Telemetry — what's disabled and why

DeepSeek Harness ships a built-in telemetry plugin
(`dsh-session-telemetry-otel`) controlled by two environment variables:

| Variable | Effect |
| --- | --- |
| `DSH_TELEMETRY_MODE` | `FULL` streams every session event as OTLP logs to DeepSeek's endpoint. `FEEDBACK_ONLY` (the project's own default) uploads nothing until you explicitly run `/feedback`. `DISABLED` keeps everything local. |
| `DSH_TELEMETRY_DISABLED` | A **hard opt-out**: any non-empty value (including `"0"` or `"false"`) disables telemetry unconditionally — it wins over `DSH_TELEMETRY_MODE` and any project-level `.env` loaded later. |

This kit disables telemetry two ways, at two different layers:

1. **Image-level env vars** (`Dockerfile`):
   ```dockerfile
   ENV DSH_TELEMETRY_DISABLED=1 \
       DSH_TELEMETRY_MODE=DISABLED
   ```
   `DSH_TELEMETRY_DISABLED=1` alone is authoritative per the project's own
   docs; `DSH_TELEMETRY_MODE=DISABLED` is set alongside it for clarity.

2. **Network policy** (`spec.yaml`):
   ```yaml
   permissions:
     network:
       deny:
         - harness-telemetry.deepseeksvc.com
   ```
   Even if the env-var opt-out were ever bypassed by a bug, the sandbox
   still can't reach the collector — a network-layer guarantee, not just
   an application setting.

This only covers DeepSeek's own built-in telemetry seam — it has no effect
on third-party observability plugins (OTel/Langfuse exporters, etc.) if you
choose to install one yourself.

## Notes

- DeepSeek Harness is an active developer preview; expect breaking
  changes. Consider pinning `@deepseek-ai/dsh@<version>` in the `Dockerfile`
  instead of the floating `latest` npm tag once you've settled on a
  version to test against.
- No `testdata/tck.yaml` is included — that fixture is only needed if
  you're contributing this kit upstream to `docker/sbx-kits-contrib`'s own
  test suite, not for using it on your own.
- `sandbox.image` points at `ghcr.io/jlugagne/dsh-sbx-kit:latest`, published
  by this repo's `build-dsh.yml` workflow.
