# Design: UI-Agent Eval Harness

Status: draft for Milestone 0. Decisions marked **[open]** are not yet settled and will be updated here as they are made.

## 1. Problem and goals

LLM-driven browser agents are usually shown through single successful runs. That hides the properties that matter in practice: how often the agent succeeds, how stable that is across repeated runs, how many steps and how much money a success costs, and which kinds of UI trip it up.

**Goals**

1. Measure task success, efficiency, cost, latency and run-to-run variance for browser agents on UI tasks.
2. Use deterministic, programmatic success checks so scores are reproducible and auditable.
3. Explain failures through a categorized taxonomy, not just a success rate.
4. Make experiments reproducible with one command, with pinned models, versions and seeds.

**Non-goals**

- Not a general web benchmark; tasks target a small set of enterprise-style UI patterns.
- Not a new agent architecture; the agent loop is deliberately simple so that differences come from the model, prompt and observation mode.
- No claims about agents in general beyond what the data supports.

**Research questions**

- RQ1: How does success rate vary across component types (e.g. shadow-DOM menus vs light-DOM forms) and difficulty tiers?
- RQ2: How much does the observation mode (accessibility tree, simplified DOM, screenshot, both) change success and cost?
- RQ3: How flaky are agents when the same task is repeated, and what drives the flakiness?
- RQ4: What are the dominant failure categories?

## 2. Architecture

```
tasks/*.yaml --> [Task loader + schema validation]
                        |
apps/ (Lit test pages + mock backend) <--> [Playwright runner] <--> [Agent loop] <--> [LLM adapter]
                        |                           |
                  [Success checker]          [Trace logger: JSONL + screenshots]
                        |                           |
                        +----------> results/ (raw JSON/CSV) ----> [Analysis] ----> report + dashboard
```

Components

| Component | Responsibility |
|---|---|
| Task loader | Parses YAML tasks, validates them against a schema (zod) |
| Test apps | Small Lit / Web Component pages with tricky UI behavior, exposing checkable state |
| Mock backend | Tiny server that records requests so success can be verified server-side |
| Runner | Launches Playwright, executes agent actions, enforces step, time and token limits |
| Agent loop | Observe, ask the LLM for an action, validate it, execute it, repeat until `done` or a limit |
| LLM adapter | Provider-agnostic interface; a scripted MockLLM replays actions for tests and CI |
| Success checker | Deterministic pass/fail from DOM, URL, storage or backend log |
| Trace logger | One JSONL record per step plus screenshots: action, observation, tokens, latency, cost, error |
| Analysis | Computes statistics and produces tables, figures and the failure breakdown |

## 3. Test applications

Built by me with Lit and Web Components; no code, names or data from any employer. Candidate components:

- Searchable select with keyboard navigation
- Menu that opens relative to its trigger and dismisses on scroll or outside click
- Split button (primary action plus dropdown)
- Paginated, sortable, filterable table whose page size depends on container height
- Modal with focus trap; multi-step form with validation; toast notifications
- Shadow-DOM-heavy widgets, theme tokens, i18n strings
- Distractors: overlays, cookie banners, delayed rendering, duplicate labels

Each page exposes its state to the checker through a data attribute, `localStorage`, or a request to the mock backend.

## 4. Task specification

```yaml
id: select-search-001
goal: "In the Country dropdown, choose 'Norway' and submit the form."
start_url: /apps/select/basic
max_steps: 12
difficulty: easy          # easy | medium | hard
component_type: select
optimal_steps: 4          # measured by a human run
tags: [light-dom, keyboard]
success_check:
  type: backend_log       # dom_state | url_matches | storage_equals | backend_log
  params:
    endpoint: /api/submit
    body_contains: { country: "Norway" }
```

Rules for task quality

- Every success check must **fail on the untouched page** (a negative control is part of the test suite).
- A human completes every task once to record `optimal_steps`.
- Difficulty tiers: easy (1-3 steps), medium (4-8), hard (9+ steps or recovery from a distractor or error).
- About 20% of tasks are held out and never used when tuning prompts.

## 5. Agent loop

- **Observation modes** (an ablation axis): accessibility tree, simplified DOM, screenshot, screenshot plus accessibility tree.
- **Action space:** click, type, select, press key, scroll, wait, navigate, done(answer). Every action is validated; invalid actions are counted.
- **Limits:** max steps, per-step timeout, total token budget.
- **Determinism:** temperature 0 by default; seeds recorded where supported; exact model identifiers and prompt hashes stored with every run.

## 6. Metrics

Per run: success, steps taken, invalid actions, tokens in and out, cost (from a price table in config), wall-clock latency, failure category.

Per task x model x configuration (N >= 5 runs):

| Metric | Definition |
|---|---|
| Success rate | Successes / runs, with a 95% Wilson interval |
| pass@k | Probability that at least one of k runs succeeds (k = 3 or 5) |
| Flakiness | Share of tasks with mixed outcomes across identical runs; run-to-run variance |
| Step efficiency | `optimal_steps / actual_steps` on successful runs |
| Cost per success | Total spend / successes |
| Latency per success | Wall-clock time on successful runs |
| Recovery rate | Success on hard tasks that contain an injected distractor or error |

Aggregates: success by difficulty and component type; success by observation mode; success by prompt variant; failure-category distribution.

## 7. Evaluation methodology

- **Deterministic scoring only** for headline numbers. An LLM judge is never used to decide pass or fail.
- **Uncertainty:** report confidence intervals; claim a difference between configurations only when intervals separate or a paired test (McNemar or bootstrap over tasks) supports it.
- **Held-out set:** about 20% of tasks reserved; results reported separately to show no overfitting to the prompt.
- **Failure taxonomy:** wrong element targeted; shadow-DOM miss; premature `done`; action loop; acted before render (timing); hallucinated element; gave up; invalid action; budget exceeded. An automatic heuristic classifier is validated against a manual review of at least 30 failures, and the agreement is reported.
- **Reproducibility:** one command reruns an experiment from a config file; raw results are versioned JSON/CSV; lockfile, pinned Playwright version and a Docker image fix the environment.
- **Honesty:** negative and surprising results are reported. A number appears in the README only if it links to raw data in `results/`.

## 8. Key decisions

| Decision | Choice | Rationale |
|---|---|---|
| Scoring | Deterministic checks (DOM, URL, storage, backend log) | Reproducible, auditable, no judge bias |
| Task format | YAML with a validated schema | Easy to author and diff; errors caught early |
| Primary language | TypeScript for the runner and apps; Python for analysis **[open]** | Playwright and Lit pair naturally with TypeScript; analysis is quicker in pandas and SciPy |
| LLM access | Provider-agnostic adapter plus MockLLM | Compare models fairly; CI needs no API keys |
| Test targets | Self-built Lit pages | Controlled difficulty; no proprietary material |
| Repeats | N >= 5 per cell | Needed to estimate variance honestly |

## 9. Repository layout (planned)

```
tasks/        YAML task definitions
apps/         Lit test pages and mock backend
harness/      runner, agent loop, adapters, scoring, trace logging
analysis/     statistics, plots, failure classification
results/      raw run data and generated figures
dashboard/    static comparison UI
docs/         DESIGN.md, architecture diagram, task-authoring guide
```

## 10. Milestones

| Milestone | Deliverable | Done when |
|---|---|---|
| M0 | Design docs, scaffold | README and DESIGN merged |
| M1 | 5 components, 5 tasks, runner, MockLLM | One end-to-end test passes in CI |
| M2 | Real LLM adapter, a11y-tree observation, 15 scored tasks | First baseline stored in `results/` |
| M3 | Repeated runs and statistics, second model, 30+ tasks | Intervals and observation-mode ablation reported |
| M4 | Failure taxonomy, held-out evaluation, dashboard | Classifier agreement with manual labels reported |
| M5 | Docs, diagram, demo, Docker reproduction, write-up | A stranger can reproduce the headline table with one command |

## 11. Threats to validity

- **Small task set:** results describe these tasks, not the web at large.
- **Self-built apps:** may be easier or harder than real applications in ways I do not control for.
- **Nondeterminism:** even at temperature 0, providers can change behavior; versions are pinned and recorded.
- **Prompt sensitivity:** one prompt per variant; held-out tasks guard against overfitting but do not remove the effect.
- **Checker bugs:** a wrong success check silently corrupts scores; negative controls and a human baseline mitigate this.

## 12. Possible extension (stretch, not in scope for M1-M5)

Add "hackable" variants of tasks with deliberate loopholes (for example setting a success flag directly, calling the backend endpoint, or editing the DOM) next to "clean" variants where those shortcuts are impossible. Then measure how often agents take the shortcut and how well simple monitors detect it. This is only worth doing after the core harness produces trustworthy baselines.