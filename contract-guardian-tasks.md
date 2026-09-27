# Contract Guardian — Task List

> Derived from the [Contract Guardian Blueprint](./contract-guardian-blueprint.md).
> Tasks are ordered by build phase. Check off items as you go.

---

## Phase 0 — Setup (Hour 0–2)

- [ ] Create 3 synthetic repos (or 3 local folders simulating repos): `payments-service`, `invoice-service`, `reporting-service`
- [ ] Plant a deliberate, non-obvious contract break in `payments-service` (`calculateDiscount` can now return negative values)
- [ ] Write out the full demo story / "aha moment" narrative before writing any code
- [ ] Scaffold a minimal FastAPI or Flask webhook server that accepts `pull_request` events
- [ ] Set up `ngrok` (or equivalent) to expose the local webhook endpoint for GitHub
- [ ] Register the webhook on the test/demo GitHub repository

---

## Phase 1 — Repo Graph Indexer (Hour 2–8)

- [ ] Install and configure `tree-sitter` Python bindings for the target language (Python **or** JS — pick one)
- [ ] Write a symbol extractor that parses source files and outputs a **symbol table** (functions, exported types, public methods) with: `repo`, `file`, `name`, `kind`, `signature`, `start_line`, `end_line`, `hash`
- [ ] Write an import-resolution pass to detect cross-repo call/reference edges
- [ ] Define the SQLite schema and create the database (`symbols`, `edges`, `findings` tables)
- [ ] Persist extracted symbols and edges to SQLite
- [ ] Write a sanity-check CLI script that prints the dependency graph for a given symbol
- [ ] Verify the planted contract break shows up as an edge between `payments-service` → `invoice-service`

---

## Phase 2 — Diff Extractor & Impact Resolver (Hour 8–14)

- [ ] Set up GitHub API client (auth token, REST calls)
- [ ] Implement the **Diff Extractor**: parse incoming PR diff, map changed lines to changed symbols using the indexer's symbol table
- [ ] Filter out purely cosmetic diffs (formatting-only, comment-only changes)
- [ ] Implement the **Impact Resolver**: BFS query on the graph store to compute transitive consumer closure (depth 2–3)
- [ ] Rank consumers by proximity (direct call first) and by optional `tier: critical` repo config
- [ ] Implement the **Consumer Fetcher**: for each consumer symbol found, pull the actual call-site code (surrounding lines window) and any tests referencing the changed symbol

---

## Phase 3 — Semantic Risk Agent (Hour 14–22) ⭐ Core — Spend Most Time Here

- [ ] Design the per-consumer prompt template containing:
  - Old implementation of the changed symbol
  - New implementation of the changed symbol
  - Consumer call-site code (with context window)
  - Structured JSON output spec: `{ risk_level, explanation, assumption }`
- [ ] Implement the **Semantic Risk Agent** as a single async function that calls the LLM (Claude via Anthropic Messages API) with the above prompt
- [ ] Wire up parallel execution using `asyncio.gather` — one subagent call per consumer
- [ ] Add per-call timeout and graceful degradation (failed call → mark as `unknown`, don't block others)
- [ ] Iterate on the prompt against the planted `calculateDiscount` scenario until the model reliably returns `risk_level: "high"` with the correct assumption identified
- [ ] Test against at least one **non-breaking** change to validate false-positive rate (expect `low`/`none` result)

---

## Phase 4 — Aggregation, Ranking & PR Comment (Hour 22–28)

- [ ] Implement the **Aggregator / Ranker**: collect all subagent verdicts, sort by `risk_level` (high → medium → low) and consumer criticality
- [ ] Filter out `none`/`low` findings from the default output (make verbose mode optional)
- [ ] Implement the **PR Comment Composer**: render findings as a single Markdown comment with consumer file/line, the specific assumption at risk, and evidence
- [ ] Post the comment to the PR via GitHub REST API (`POST /repos/{owner}/{repo}/issues/{issue_number}/comments`)
- [ ] Store findings in the `findings` SQLite table for later review
- [ ] *(Stretch)* Implement the **Patch Drafter** subagent: for the top `high` finding only, ask the LLM to generate a minimal defensive fix as a suggested diff block (collapsible in the comment)

---

## Phase 5 — Polish & Demo Rehearsal (Hour 28–34)

- [ ] Rehearse the full end-to-end demo at least **3 times** before presenting
- [ ] Prepare a fallback: cache the exact demo run's API responses for use if live calls are slow or rate-limited
- [ ] Prepare the 3-minute demo script slide or talking points (see §9 of the blueprint)
- [ ] Add the dependency graph diagram (even a static Mermaid render) for the demo screen
- [ ] Document `README.md` with setup instructions, environment variables, and how to run the pipeline
- [ ] Prepare one-sentence answers to anticipated judge questions:
  - "Why not one big prompt?" → Context dilution, parallelism, isolated failure (§3)
  - "How does this scale to 10,000 symbols?" → Incremental indexing + LLM only runs on transitive-closure subset per PR (§8)
  - "Is this safe?" → Advisory only, never auto-merging, human-in-the-loop always (§6)

---

## Stretch Goals (cut without shame if time is short)

- [ ] Patch Drafter for all `high` findings (not just the top one)
- [ ] Support a second language in the indexer (proves generality)
- [ ] Static Mermaid graph visualization of the consumer chain embedded in the PR comment
- [ ] Confidence calibration: run against several more planted scenarios and report a true-positive precision number

---

## Explicitly Out of Scope (do not build)

- Multi-language support beyond 1–2 languages
- Auto-blocking or auto-merging PRs
- Full behavioral inference for dynamically-typed languages via static analysis
- Polished web dashboard or UI

---

## Validation Metrics to Track / Demo

| Metric | Target |
|---|---|
| Planted `high` risk break correctly flagged | ✅ must pass |
| Non-breaking change returns `low`/`none` | ✅ must pass |
| Wall-clock time: PR open → comment posted | Report actual time |
| False-positive rate | As low as possible; demo at least 2 non-breaking cases |
| Shift-left: break caught pre-merge vs. production | Core story for pitch |
