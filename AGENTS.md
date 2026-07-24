# TurboFieldfare

Swift and Metal inference for Gemma 4 26B-A4B on Apple Silicon.

## Scope

This checkout is for running and reporting existing behavior. Do not edit source, change runtime defaults, or start optimization work unless the user asks.

## Layout and commands

`Sources/TurboFieldfare/` is the runtime; `Sources/TurboFieldfareRepack/`, `Sources/TurboFieldfareCLI/`, and `Sources/TurboFieldfareApp/` contain the installer, CLI, and Mac app.
`Tests/` contains focused public tests; `docs/` contains design, benchmark, and experiment notes.

```bash
swift run -c release TurboFieldfareRepack --output scratch/gemma4.gturbo
swift build -c release
.build/release/TurboFieldfareMac
swift run -c release TurboFieldfareCLI \
  --model scratch/gemma4.gturbo \
  --prompt "The capital of France is" \
  --max-new 64
```

The installer streams the pinned model without staging the full source checkpoint. Set `HF_TOKEN` only if requested. The download is about 15 GB.

## Test rules

Before a model run, require macOS 26+, Swift 6.2+, enough disk, acceptable `memory_pressure -Q`, a completed `scratch/gemma4.gturbo`, and no process from `pgrep -fl 'TurboFieldfareMac|TurboFieldfareDecodeService|TurboFieldfareCLI|TurboFieldfarePackageTests|swiftpm-testing-helper|mlx_lm|mlx-lm'`. If a check fails, inform the user and stop; do not terminate apps or delete or reinstall the model.

Run package tests through `Scripts/test.sh`. Run only one app, CLI, or model-using test at a time.

For performance results, build release once and follow the [community benchmark guide](docs/COMMUNITY_BENCHMARKS.md) exactly. Do not enable experimental controls or profiling.

Do not download a full checkpoint, duplicate the `.gturbo` model, create a worktree, or purge caches just to run tests.

Report the commit, hardware and RAM, macOS, Swift version, exact command, exit code, complete timing footer or error, and every protocol deviation. Treat results as measurements, not performance ceilings.

## App controls

The Mac app sends prompts through the pinned Gemma 4 IT chat format. It
exposes context length, temperature, Top-K, Top-P, expert-cache slots, prefill,
and RDADVISE. The defaults are temperature `0.2`, Top-K `64`, and Top-P `0.95`.
Responses can use the context space left after formatting the prompt, and FP16
is the runtime KV format. The HUD shows generation rate, token count, and
decode-service memory; Last run also shows time to first token and I/O. Build
the app with its sibling `TurboFieldfareDecodeService`; it never loads a second
in-process model. See [README](README.md) and [Runtime controls](docs/RUNTIME_CONTROLS.md).

<!-- s5d:begin -->
## S5D — Agentic Change Control Plane

This repo uses **S5D** (https://github.com/system5-dev/s5d) to describe target state, record agent/tool evidence, bind architectural decisions, and verify that code still matches them.

**⛔ S5D is MANDATORY for non-trivial work.** Architectural decisions, new features, refactors >30 LOC, and any change touching multiple modules MUST go through the S5D flow before implementation. Skip ONLY for: bug fixes <30 LOC, config-only, docs-only. `S5D_BYPASS=1` is an explicit break-glass escape hatch, not routine flow; document the justification when you use it. When in doubt, run `s5d_route` to classify the request.

**Flow:** target state → edit spec → `s5d_validate` → `s5d_preview` → `s5d_approve` → run/implement → `s5d_run_gates` → `s5d_import` → `s5d_drift_check`.

**Outcome protocol.** When an approved spec declares `workflow.outcome`, treat a native host goal as a session projection only. The source of truth is `s5d outcome check <spec>` over current, source-bound gate evidence; the Stop hook enforces that verdict only on hosts that support blocking. Never auto-run authority transitions such as approval, phase acceptance, mandate admission, or waiver — they require current human authorization.

**MCP tools** (prefer over shell CLI when available):
- `s5d_route` — classify a request into tier/mode/entry
- `s5d_codebase_sync` / `s5d_codebase_check` — codebase coverage snapshot
- `s5d_discover_sync` / `s5d_discover_check` — discovery graph snapshot
- `s5d_check` — architecture check (components vs. source paths)
- `s5d_new` / `s5d_note` — create spec / quick note
- `s5d_validate` / `s5d_preview` — dry-run checks before approval
- `s5d_outcome_check` — read the shared source-bound outcome verdict
- `s5d_approve` / `s5d_import` — commit decision, bind SHA256 chain
- `s5d_drift_check` / `s5d_reconcile` / `s5d_rollback` — verify & recover
- `s5d_show` / `s5d_status` — inspect specs and project state

**Commits reference specs.** When a change is governed by an S5D spec, include `S5D-Spec: <spec-id>` as a trailer in the commit body (e.g. `S5D-Spec: feat.s5d.structure-outline-and-vertical-phases`). This binds the commit to the decision record and lets `git log --grep='S5D-Spec:'` reconstruct the architectural rationale. Trivial changes that skipped S5D need no reference.

Specs live in `.s5d/packages/`. Run `s5d --help` or read `skills/s5d/SKILL.md` for full reference.
<!-- s5d:end -->
