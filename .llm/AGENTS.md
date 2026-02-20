# FullSynth Repo Instructions (Source of Truth)

This file is the canonical repo-scoped instruction file.

Source-of-truth policy:
- Canonical file: `./.llm/AGENTS.md`
- Wrapper entrypoints: `./AGENTS.md`, `./CLAUDE.md` (symlinks)
- Keep detailed execution spec in `./INSTRUCTIONS.md`
- If guidance conflicts, follow the stricter requirement.

## Project Goal

Build a reproducible Songsterr data-prep pipeline that extracts high-quality **single-guitar** segments (tab + sync + stem aligned) for downstream model training.

## Current Execution Contract

- Work autonomously without stopping unless blocked by credentials/access failures.
- For this kickoff run, continue uninterrupted until **Friday, February 20, 2026, 12:45 PM Europe/Madrid**.
- If core scope finishes early, continue with highest-impact next steps (validation, robustness, quality filters, docs, and future-improvement backlog).
- If started after that cutoff time, continue for at least 120 minutes from start.

## Required Safety Rules

- Stage DB only via `srdb`.
- First DB command must be `srdb profiles`; confirm `stage [default]`.
- Never expose secrets/config contents from `~/.config/songdb/config.toml`.
- Do **not** run parallel `srdb` queries (tunnel port collisions).
- Prefer bounded queries, aggregates, and intermediate artifacts.

## Pipeline Scope (See `INSTRUCTIONS.md` for full details)

Implement and maintain CLI stages:
1. `discover`
2. `index-segments`
3. `materialize-sample`
4. `qc-report`
5. `full-run` (remote on `hyper3`)

Default targeting:
- Top 10k songs by `Song.guitarViews`
- Guitar instrument allowlist starts with `{24,25,26,27,28,29,30}`
- Exclude bass/non-guitar from single-guitar segment logic
- Choose best `VideoPoints` candidate per revision using guitar-stem and sync quality ranking

## Continuity and Memory (Mandatory)

Store reusable knowledge in `.llm/` during execution:

- Session logs: `.llm/logs/YYYY-MM-DD.md`
- Process feedback pointers: `.llm/memory/process-feedback-YYYY-MM-DD.md`
- Ongoing improvements backlog: `.llm/memory/future-improvements.md`

Rules:
- Append-only; never overwrite prior entries.
- Record concrete errors and exact fixes.
- Record data quality issues + mitigation ideas.
- Record architecture/design decisions and tradeoffs.
- Keep notes concise, grep-friendly, and actionable.

## Delivery Expectations

Always keep outputs reproducible and resumable, with explicit commands and artifacts documented in `README.md`.
