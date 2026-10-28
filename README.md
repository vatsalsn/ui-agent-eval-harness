# UI-Agent Eval Harness

An open-source framework for measuring how reliably LLM-driven browser agents complete realistic UI tasks, and for finding out *where and why* they fail.

> **Status: Milestone 0 of 5 (design).** This repository currently contains the design only. No code runs yet and **no results have been measured**. The checklist below is updated as each piece lands.

## Why this exists

Browser-agent demos are usually anecdotes: one task, one run, one success. That tells you little about reliability. This project aims to answer a narrower, measurable question:

> On enterprise-style UI components (dropdowns, menus, split buttons, paginated tables, modals, shadow-DOM widgets), how often does an agent finish the task, how efficiently, at what cost, and what breaks it?

## How it will work

1. **Tasks** are defined in YAML: a natural-language goal, a start URL, a step budget, and a programmatic success check.
2. **Test apps** are small Lit / Web Component pages built for this repo, each with deliberately tricky UI behavior (scroll-dismissed menus, keyboard navigation, delayed rendering, overlays, duplicate labels).
3. **The runner** drives a Playwright browser. An agent loop observes the page, asks an LLM for the next action, and executes it.
4. **Scoring** is deterministic: DOM state, URL, storage, or requests recorded by a mock backend. No LLM judge decides pass or fail.
5. **Every run is traced**: actions, observations, screenshots, tokens, latency, cost, errors.
6. **Experiments repeat runs** (N >= 5 per task, model and config) so variance and flakiness are measured, not guessed.
7. **Reports** aggregate success rates with confidence intervals, failure categories, and ablations across models, prompts and observation modes.

See [docs/DESIGN.md](docs/DESIGN.md) for the full design, metrics and evaluation methodology.

## Planned features

- [ ] Task spec schema and validator (YAML + zod)
- [ ] Lit / Web Component test apps (5 for M1, 15+ by M3)
- [ ] Playwright runner with full trace capture
- [ ] Deterministic success checkers (DOM, URL, storage, backend log)
- [ ] Provider-agnostic LLM adapter, plus a scripted MockLLM for tests and CI
- [ ] Observation modes: accessibility tree, simplified DOM, screenshot, screenshot + tree
- [ ] Repeated-run statistics: Wilson confidence intervals, pass@k, flakiness
- [ ] Failure taxonomy with a manually validated sample
- [ ] Static comparison dashboard
- [ ] Dockerized, one-command reproduction

## Results

**Not yet measured.** Numbers will appear here only when they are backed by raw data committed under `results/`.

## Quickstart

TODO. Added when the first runnable milestone lands.

## Adding a task

TODO. Documented once the task schema is stable.

## Adding a model adapter

TODO. Documented once the adapter interface is stable.

## Limitations (known in advance)

- The task set will be small, and the test apps are my own, so they may not represent every real application.
- LLM outputs are nondeterministic; results depend on model versions and prompts. Versions are pinned and recorded with every run.
- Findings will describe these tasks and models, not browser agents in general.

## Roadmap

| Milestone | Goal |
|---|---|
| M0 | Design docs, repository scaffold (this commit) |
| M1 | 5 test components, 5 tasks, Playwright runner, MockLLM, one passing end-to-end test |
| M2 | Real LLM adapter, accessibility-tree observation, deterministic scoring for 15 tasks, first baseline |
| M3 | Repeated runs and statistics, second model, observation-mode ablation, 30+ tasks |
| M4 | Failure taxonomy with manual validation, held-out evaluation, dashboard |
| M5 | Docs, architecture diagram, demo, Docker one-command reproduction, write-up |

## License

MIT (see `LICENSE`).

## Author

Vatsal Singhal. Feedback and issues are welcome.