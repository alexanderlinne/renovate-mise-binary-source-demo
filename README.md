# Renovate `binarySource=mise` demo

This is a small, public reproduction repository for
[renovatebot/renovate#39947](https://github.com/renovatebot/renovate/issues/39947): support for
`binarySource=mise` in Renovate, which lets Renovate install the tools needed for artifact
updates (here, a Go toolchain) from the repository's own `mise.toml`, instead of relying on a
tool already present on `PATH`.

A single manually triggered workflow run demonstrates, end to end:

1. **Control:** with `binarySource=global`, Renovate cannot update `go.sum` because no usable
   `go` is on `PATH` (the runner's own Go is deliberately disabled for this demo, see below).
2. **mise:** with `binarySource=mise`, Renovate installs Go via mise, updates `go.sum`, and opens
   a pull request.
3. **mise's Go was used:** the runner's (stubbed) `go` was never invoked during the mise run.
4. **Safe mode:** the repository's `mise.toml` could not execute arbitrary code. It contains a
   harmless canary (`[env] MISE_DEMO_CANARY`) that would `touch` a file under `/tmp` if mise
   evaluated it outside of safe mode — the workflow asserts that file is never created.

The workflow checks out Renovate from a fork/branch implementing the feature
(`alexanderlinne/renovate@feat/39947-mise-binary-source`) and runs it from source
(`node lib/renovate.ts`), since the feature is not in any published Renovate release.

## How to run it

1. Go to **Actions → "Demo: mise binarySource" → Run workflow**.
2. Wait for the run to finish, then read the job's log directly in the Actions UI (both Renovate
   runs log at debug level to the console, so everything needed to check the claims above —
   `Failed to update go.sum` / `Returning updated go.sum`, `installing mise tools and fetching
   mise environment variables` with `safeMode: true`, the mise `PATH` entry under
   `.../mise/installs/go/...`, and the `Executing command` entries for `go ...` — is right there
   in the run log).
3. The "Print and reset Go stub log" and "Print Go stub log ... and safe-mode canary status"
   steps print, in plain text, whether the runner's (stubbed) `go` was invoked during each run
   and whether the safe-mode canary file was created.
4. Check the repository's **Pull requests** tab for the PR updating `github.com/pkg/errors` to
   `v0.9.1`.

No secret is required: the workflow uses the default `GITHUB_TOKEN` with
`contents: write` and `pull-requests: write` permissions (declared on the job), so Renovate can
push the update branch and open the PR itself.

### Re-running

Renovate won't open a second PR for an update it has already proposed. To get a fresh PR on the
next run, close the existing PR and delete its branch first. If you skip this, the run still
passes (an existing open PR satisfies the check), but you won't see a new PR.

## Repository contents

- `go.mod`, `go.sum`, `main.go` — a minimal Go module with one outdated dependency
  (`github.com/pkg/errors v0.9.0`, archived upstream, so the update target
  `v0.9.1` is stable) for Renovate's `gomod` manager to update.
- `mise.toml` — declares `go = "1.26"` as the tool Renovate should install via mise, plus the
  safe-mode canary described above.
- `renovate.json` — restricts Renovate to the `gomod` manager and this one dependency, so the
  demo produces exactly one predictable PR.
- `.github/workflows/demo-mise-binary-source.yml` — the one workflow described above.
