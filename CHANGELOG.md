# Changelog

Notable changes, newest first. Releases are git tags; this file says what each tag contains.

## v0.1.1

A security release. **The service now builds with Go 1.27.2**, and its libraries move to their Go 1.27.2
releases. The configuration, the cache layout it writes and the trust it serves are unchanged.

### Changed

- **Go 1.27.2** (was 1.27.0), both in `go.mod` and as the image's default build toolchain
  (`ARG GO_VERSION` in the `Dockerfile`). Go 1.27.2 fixes standard-library and `golang.org/x/net`
  vulnerabilities that this service's code reaches under 1.27.0: `govulncheck` found eleven it calls,
  GO-2026-6599, -6600, -6603, -6605, -6607, -6608, -6610, -6611, -6612, -6613 and -6617, in
  `html/template`, `net/http`, `crypto/tls`, `net/textproto`, `mime/multipart` and `golang.org/x/net`.
  Under the new set it finds none. **Rebuild the image to pick the fixes up**; an image built before
  this change still carries them.
- **Libraries:** `go-eudi-trust` → v0.1.3, `go-verifier-helpers` → v0.0.5, `go-platform-kit` →
  v1.11.4, `go-authbyte` → v0.29.1 (from v0.23.1), azugo → v0.40.0, `valkey-go` → v1.0.78, `fasthttp`
  → v1.75.0, `VictoriaMetrics/metrics` → v1.44.1, `miniredis` (tests only) → v2.40.0, plus 38 indirect
  modules with them. The go-authbyte releases in between are additions this service does not use,
  with one visible change: when the authenticated call to trust-anchor is refused, the error no
  longer repeats the answer's body. The trust-list fetcher uses its own HTTP client with its own
  timeout, so azugo's new deadline handling does not reach it; nothing else in azugo v0.40.0 does
  either (no cookies, redirects or azugo cache of its own).

### Notes

- `LICENSE` drops the boilerplate header that preceded the licence text (merged from the published
  repository). The licence itself is unchanged: EUPL-1.2.
- The image build now caches Go modules and compiled packages between builds. The binary is built by
  the same command with the same flags; only the build's layer order and caches changed.
- CI's linter moved to golangci-lint v2.14.0. The earlier release cannot read Go 1.27.2's compiled
  standard library and stops before linting anything.
- The gate is green on Go 1.27.2, with the `testhelpers` build tag as CI uses it: `go mod verify`,
  `go mod tidy -diff`, build, vet, `gofmt`, golangci-lint v2.14.0, `go test -race` with **0 races**,
  and the fuzz target for 30 seconds with no failing input; `govulncheck` reports **no
  vulnerabilities found**.

## v0.1.0 — initial public release

**The background worker that keeps the fleet's trust cache warm: the only component that talks to the trust service.**

It materialises trust anchors, snapshot freshness and the hottest status lists into a shared cache,
fail-closed. Consumers read that cache and never fetch trust data themselves, so one warm cache absorbs
the whole fleet's trust reads, and a withdrawal takes effect for every reader the moment a new snapshot
is materialised — a per-type atomic replacement, so a reader observes strictly before- or
strictly after-swap state.

It exposes probes and metrics plus a key-gated operator surface; without the key that surface is
unbound.

### Running it

The image entrypoint is `["/server", "web"]`, with `["/server", "health"]` as the healthcheck
command — a compose or orchestrator file that also passes `command: ["web"]` would append a second
argument. Every setting is an environment variable with a safe default where one is possible; the
README lists them, and any secret can be supplied as `<NAME>_FILE` pointing at a mounted file instead.

### What it needs

Go **1.27.0** to build. Libraries at this release:

`go-eudi-trust` v0.1.2 · `go-verifier-helpers` v0.0.4 · `go-authbyte` v0.23.1 ·
`go-platform-kit` v1.11.3 · `azugo.io/azugo` + `azugo.io/core` v0.38.1
