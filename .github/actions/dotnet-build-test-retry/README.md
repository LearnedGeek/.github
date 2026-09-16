# dotnet-build-test-retry

Composite GitHub Action that runs `dotnet restore`, `dotnet build`, and
`dotnet test` for a solution, with a **targeted** retry loop around the
MSB4166 "Child node exited prematurely" MSBuild worker crash. That error
is a self-hosted-runner resource-crunch flake (typically parallel worker
OOM under memory pressure), not a real code failure. Every other build
failure fails fast on the first attempt.

## Why this exists

Self-hosted runners for LearnedGeek projects occasionally see MSB4166
mid-build. It's a known intermittent — reruns of the exact same commit
pass. Manually clicking "Re-run failed jobs" works but wastes real
time, especially on production deploys with a stakeholder waiting.

This action bakes the retry into a single reusable step so every
consumer benefits from the same fix + can iterate on the detection
logic in one place.

## Usage

```yaml
- uses: LearnedGeek/.github/.github/actions/dotnet-build-test-retry@v1
  with:
    solution: src/YourApp.slnx
    version: ${{ steps.version.outputs.version }}
    test-filter: FullyQualifiedName!~YourApp.E2E   # optional
```

## Inputs

| Input | Required | Default | Purpose |
|---|---|---|---|
| `solution` | yes | | Path to the `.sln` / `.slnx` file. |
| `configuration` | no | `Release` | MSBuild configuration. |
| `version` | yes | | Passed to `-p:Version=` on build. |
| `test-filter` | no | `""` | `dotnet test --filter` argument. Empty = no filter. |
| `max-attempts` | no | `2` | Max build attempts. Only MSB4166 consumes an attempt. |
| `backoff-seconds` | no | `30` | Sleep between attempts. |
| `restore-args` | no | `""` | Extra `dotnet restore` flags. |
| `build-args` | no | `""` | Extra `dotnet build` flags. Escape hatch: pass `-m:1` if MSB4166 becomes chronic on a specific caller. |
| `upload-logs` | no | `"true"` | Upload per-attempt build logs as an artifact for post-mortem. |

## Retry behavior

For every build attempt the stdout+stderr are captured to
`build-attempt-N.log`. On failure:

- If the log contains `MSB4166` **and** attempts remain → sleep
  `backoff-seconds` and retry.
- Every other failure (compile error, analyzer error, missing package)
  → fail immediately.

## Versioning

Consumers pin `@v1` (rolling major-version tag). New `v1.x.y` releases
get picked up automatically on the next workflow run — no downstream
PR needed.

- **`v1.x.y`** immutable specific tags cut on every release
- **`v1`** rolling tag moved to point at the newest `v1.x.y`
- **`@v2`** would only exist if we ever ship a breaking change (renaming
  an input, removing an output, etc.). Downstreams migrate on their
  own timing at that point.

Non-breaking changes (add optional input, broaden retry detection,
improve log messages) stay on `v1` and ripple to all consumers
automatically.

## Consumer examples

**HuntApp (`deploy-stg.yml` / `deploy-prod.yml`)** — replaces the raw
`dotnet restore/build/test` step with a single `uses:` block, calling
into `LearnedGeek/.github/.github/actions/dotnet-build-test-retry@v1`.

## Local development / testing this action

There's no local runner for GitHub Actions composite actions, but the
shell logic is straightforward. To smoke-test the retry loop
end-to-end, point a workflow at a feature branch of this repo
(`@feat/dotnet-build-test-retry`) and dispatch it against a solution
that reliably triggers MSB4166 on your self-hosted runner. Once the
retry visibly kicks in, cut the release tags.

## Rollout checklist

- [ ] Merge this PR to `main`
- [ ] Cut `v1.0.0` immutable tag on `main`
- [ ] Create/move rolling `v1` tag → `v1.0.0`
- [ ] Push both tags
- [ ] Migrate HuntApp `deploy-stg.yml` + `deploy-prod.yml` (canonical example)
- [ ] Migrate allevo / lakecountryspanish / mcarthey.com deploy workflows
