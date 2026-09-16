# go-lint-kit

An sbx kit (Docker Sandboxes) that adds to an agent:

- **Go** — latest version, installed into `~/.local/go`
- **[golangci-lint](https://golangci-lint.run/)** — Go meta-linter
- **[go-arch-lint](https://github.com/fe3dback/go-arch-lint)** — architecture linter (import rules)
- **[diffity](https://github.com/nilbuild/diffity)** — diff viewer / code review tool

## Usage

Locally:

    sbx run --kit ./go-lint-kit claude

From a Git repo (once pushed to GitHub):

    sbx run --kit "git+https://github.com/JLugagne/sbx-kit.git#dir=go-lint-kit" claude

Pin a tag/commit for production use:

    sbx run --kit "git+https://github.com/JLugagne/sbx-kit.git#ref=v1.0.0&dir=go-lint-kit" claude

## Extending a specific agent

By default this kit is a generic `mixin` (`shell` template). To attach it
explicitly to a specific agent (e.g. Claude), add to `spec.yaml`:

    extends: claude

## Testing the kit

    cd go-lint-kit
    ../scripts/test-kit.sh        # TCK (unit test, containerized)
    ../scripts/test-kit-e2e.sh    # e2e (real sbx sandbox, requires sbx + Docker Hub login)

(scripts provided by the `docker/sbx-kits-contrib` repo — copy them over if
you place this kit in your own repo and want to reuse their TCK.)

## Notes

- The `install` commands run as user `1000` (non-root): Go is therefore
  installed into `$HOME/.local/go` rather than `/usr/local/go`, and
  `PATH`/`GOPATH` are declared via `environment.variables`.
- `network.allowedDomains` lists **every** domain reached by the installs
  (Go, the golangci-lint script + its GitHub release, the Go module proxy
  for `go install`, and the npm registry for `diffity`). Under a
  `deny-all` policy, a missing domain will make `sbx create` fail.
- `npm` is assumed to already be present in the agent's base image (true
  for Node-based CLI agents like Claude Code/Codex); otherwise, add a
  Node.js install step before the diffity one.
