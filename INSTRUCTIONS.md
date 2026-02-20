# Autonomous Execution Instructions (FullSynth)

You are an autonomous engineering agent. Execute end-to-end without waiting for user input unless a hard credentials/access blocker appears.

## Hard Runtime Guard

- Start immediately and work continuously without pausing/stopping until **Friday, February 20, 2026, 12:45 PM Europe/Madrid**.
- Do not stop early just because the initial milestone is complete.
- If initial scope finishes before 12:45 PM, immediately continue with the next highest-impact tasks (validation, scaling, robustness, documentation, and follow-up improvements).
- If current time is already past 12:45 PM Europe/Madrid at launch, run uninterrupted for at least 120 minutes from start.

## Mission

Build a reproducible pipeline that produces a high-quality dataset index of **single-guitar segments** from Songsterr data, using:

- top songs by guitar popularity
- tab tracks + sync points + separated stems
- measure-level logic to find stretches where exactly one guitar track is active

Primary outcomes:

1. robust discovery/indexing pipeline
2. segment manifests (parquet + csv preview)
3. sample materialized audio clips for QC
4. quality report + yield funnel
5. remote execution flow on `hyper3`

## Mandatory Safety/Workflow Constraints

- Use `srdb` safely and stage-only.
- First DB command must be: `srdb profiles` and must show `stage [default]`.
- Do not print secrets from `~/.config/songdb/config.toml`.
- Do not run `srdb` queries in parallel (fixed local tunnel port can collide).
- Prefer aggregates and bounded queries with `LIMIT` for exploration.
- Build idempotent/resumable steps and checkpoint artifacts.

## Context

- Data source is Songsterr staging DB and public Songsterr/CloudFront assets.
- Focus set: top 10k songs by `Song.guitarViews`.
- Keep only guitar tracks, exclude bass/non-guitar tracks for single-guitar detection.
- Songs can have multiple guitar tracks; the target is **segments where exactly one guitar track is active**.
- Use remote server alias `hyper3` for full run.

## Repo Requirements

Create/maintain this structure:

- `pyproject.toml`
- `README.md`
- `configs/`
- `scripts/`
- `src/sgprep/`
- `tests/`
- `artifacts/` (generated)
- `.llm/logs/`
- `.llm/memory/`

Suggested Python deps:

- `polars`, `duckdb`, `pydantic`, `requests` or `httpx`, `orjson`, `typer`, `tqdm`, `tenacity`, `soundfile`, `numpy`

External dependency:

- `ffmpeg`

## Pipeline Phases (separate CLI commands)

1. `discover`
- Build candidate song/revision/track/video-point tables from stage DB.
- Write outputs under `artifacts/discovery/`.

2. `index-segments`
- Download/parse tab JSON for relevant tracks.
- Compute per-measure track activity.
- Find contiguous runs where exactly one guitar track is active.
- Map measure runs to time via sync points.
- Emit raw and filtered segment manifests.

3. `materialize-sample`
- Download stem audio for QC subset.
- Extract clips into local files.

4. `qc-report`
- Produce report with counts, durations, yield funnel, rejection reasons, sample table.

5. `full-run`
- Run full pipeline on `hyper3`, checkpointing all phases.

## SQL Templates

### Top songs

```sql
SELECT "songId", "guitarViews", "title", "artist"
FROM "Song"
WHERE "deletedAt" IS NULL
  AND "hasPlayer" = true
  AND "hasGuitarTrack" = true
  AND COALESCE("isJunk", false) = false
ORDER BY "guitarViews" DESC NULLS LAST
LIMIT 10000;
```

### Latest usable revision per song

```sql
SELECT DISTINCT ON (r."songId")
  r."songId", r."revisionId", r."image", r."isRendered"
FROM "Revision" r
JOIN top_songs ts ON ts."songId" = r."songId"
WHERE r."deletedAt" IS NULL
  AND r."isBlocked" = false
ORDER BY r."songId", r."isRendered" DESC, r."revisionId" DESC;
```

### Tracks

```sql
SELECT
  t."songId", t."revisionId", t."trackId", t."instrumentId", t."name",
  t."views", t."isEmpty", t."totalMeasures", t."emptyMeasures"
FROM "Track" t
JOIN latest_revisions lr ON lr."revisionId" = t."revisionId";
```

### VideoPoints candidates

```sql
SELECT
  lr."songId", lr."revisionId", vp.id AS "videoPointsId",
  vp.status, vp.points, vp."pointsUncertain", vp."separatedAudioKeys"
FROM latest_revisions lr
JOIN "RevisionToVideo" rv
  ON rv."revisionId" = lr."revisionId" AND rv."deletedAt" IS NULL
JOIN "VideoPoints" vp
  ON vp.id = rv."videoPointsId" AND vp."deletedAt" IS NULL
WHERE vp.status = 'done';
```

## Instrument Filtering Policy

Start with:

- `GUITAR_INSTRUMENT_IDS = {24,25,26,27,28,29,30}`
- `BASS_INSTRUMENT_IDS = {33,34,35,36,38}`

Then self-validate with diagnostics by `instrumentId` and track name keyword rates (`guitar`, `bass`). If needed, adjust and document rationale.

## Candidate VideoPoints Ranking

A revision can have multiple linked VideoPoints. Rank and choose best by:

1. has `_guitar.` URL in `separatedAudioKeys`
2. `status='done'`
3. `pointsUncertain` empty (`NULL`, `''`, `'[]'`)
4. parseable `points` with length >= 8
5. highest `videoPointsId` as tiebreaker

For this first dataset pass, skip candidates without `_guitar.` stem URL.

## Tab JSON Retrieval

Try domains in order:

- `dqsljvtekg760`
- `d34shlm8p2ums2`
- `d3cqchs6g3b5ew`

URL format:

- `https://{domain}.cloudfront.net/{songId}/{revisionId}/{image}/{trackId}.json`

Handle compressed responses.

## Measure Activity Logic

For each guitar track and measure:

- measure is active if any non-rest note exists in any beat in any voice
- otherwise inactive

Per measure index in the song timeline:

- count active guitar tracks
- keep measures where count == 1 and store owner `trackId`

Build contiguous runs with same owner track:

- configurable `min_bars` (default 1)

## Time Mapping

- parse `points` as float array
- validate monotonic boundaries for used ranges
- map measure `m` start to `points[m]` (0-based)
- segment end is boundary after last measure
- clamp start to >= 0
- reject too short/long or out-of-range segments

## Required Outputs

- `artifacts/discovery/top_songs.parquet`
- `artifacts/discovery/latest_revisions.parquet`
- `artifacts/discovery/tracks.parquet`
- `artifacts/discovery/video_points_candidates.parquet`
- `artifacts/index/single_guitar_runs.parquet`
- `artifacts/index/single_guitar_segments.parquet`
- `artifacts/qc/qc_report.md`
- `artifacts/qc/yield_funnel.csv`
- `artifacts/qc/random_samples.csv`

`single_guitar_segments.parquet` must include:

- `segment_id`
- `song_id`, `revision_id`, `video_points_id`
- `track_id`, `instrument_id`
- `start_measure`, `end_measure`, `n_measures`
- `start_sec`, `end_sec`, `duration_sec`
- `stem_url_guitar`
- `points_len`, `points_uncertain_empty`
- `tab_measures_len`
- `quality_flags`
- `split`

## Remote Execution (`hyper3`)

- Ensure remote helper flow exists (`status`, `sync`, `exec`, `tmux run/tail`).
- Sync repo to `hyper3`.
- Install deps via `uv`.
- Run full pipeline in tmux.
- Keep resumability/checkpoints for each phase.

## Verification Gates

1. Dry run local (top 100 songs):
- full discover + indexing
- materialize 100-300 clips
- verify random samples

2. Full remote run (top 10k songs):
- full manifests + QC report

## Acceptance Criteria

- Pipeline runs from clean checkout with documented commands.
- Top 10k discovery completed.
- Final manifest non-empty.
- QC report includes total hours + rejection breakdown.
- At least 200 materialized clips exist.
- Outputs reproducible and resumable.

## Mandatory Continuity + Future-Improvement Memory in `.llm`

You must continuously write high-signal notes for future work in this repo under `.llm/`.

Required files:

- `.llm/logs/YYYY-MM-DD.md` (session chronology)
- `.llm/memory/process-feedback-YYYY-MM-DD.md` (error/fix and workflow gotchas)
- `.llm/memory/future-improvements.md` (evergreen backlog of improvement ideas)

Rules:

- Append, never overwrite previous entries.
- Record concrete failure signatures and exact fixes.
- Record dataset-quality issues and proposed mitigation.
- Record architecture options for next iteration.
- Keep entries short, grep-friendly, and actionable.

## If Initial Scope Finishes Early

Before stopping, continue autonomously with next steps:

1. add confidence scoring for segments
2. add stricter noise/bleed heuristics
3. add automatic spot-check script with waveform/spectrogram exports
4. add split strategy (`train/val`) minimizing artist/song leakage
5. document training-consumption interface for next model stage

## Final Report Format

At the end of execution window, provide:

1. repo path + commit hash
2. exact commands used (dry run/full run)
3. artifact paths
4. yield summary (songs, candidates, segments, hours)
5. top rejection reasons
6. top recommended next steps
