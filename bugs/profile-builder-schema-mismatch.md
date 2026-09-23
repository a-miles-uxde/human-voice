# Bug: `assemble_voice_profile()` output does not validate against `voice-profile.schema.json`

## Summary

`scoring/src/voice_scoring/profile_builder.py::assemble_voice_profile()` is meant to produce a schema-compliant Voice Profile JSON (per `agents/profile-synthesizer.md`, step "Output": *"Validate the output against `voice-profile.schema.json` before writing."*). In practice its output fails validation against `question-bank/schemas/voice-profile.schema.json` in at least six distinct ways. A profile can only be produced today by hand-transforming most of the function's output after the fact.

## Environment

- human-voice (repo, `develop` branch, as of 2026-09-23)
- Session: `3cd77a4a-59c5-4550-8c0c-ce67f4600226`
- Discovered while manually running the profile-synthesizer flow (the `human-voice:profile-synthesizer` agent type was unavailable in the session, so the work was done by hand, which is what surfaced these mismatches one validation error at a time).

## Mismatches found

All against `jsonschema.validate(profile, schema)` with `question-bank/schemas/voice-profile.schema.json`.

1. **`gold_standard_dimensions.*` shape.** `assemble_voice_profile()` emits `{score, self_report, observed, tier, weight_sr, weight_obs}`. The schema's `gold_standard_dimension` def requires exactly `{score, self_report, observed, confidence}` with `additionalProperties: false` — `tier`/`weight_sr`/`weight_obs` are rejected, and `confidence` is missing entirely.
2. **`gold_standard_dimensions` is missing dimensions when data is absent.** The schema requires all 8 keys (`formality`, `emotional_tone`, `personality`, `complexity`, `audience_awareness`, `authority`, `narrativity`, `humor`) to always be present. When a dimension's source module wasn't administered by the branching system (e.g. `audience_awareness` lives in M07, which is skipped for the `academic_technical` branch — see `bugs/` sibling issue on branching), `merged_score` is `null` and the function silently omits the key, which the schema rejects.
3. **`score`/`self_report`/`observed` can be `null` or non-integer floats.** The schema requires `integer` (not nullable) for all three. `build_profile()`'s `_merge_scores()` legitimately produces `None` for single-source-missing dimensions and floats generally (e.g. `46.41`).
4. **`calibration` shape.** `calibrate()`'s raw output (`{dimensions: {...}, overall_self_awareness: <0-100>}`) is passed straight through by `assemble_voice_profile()`. The schema requires `{overall_self_awareness: <0-1>, high_awareness_dimensions: [...], blind_spots: [...], aspirational_gaps: [...]}` — a completely different shape, and the awareness score is on a 0–100 scale instead of the required 0–1.
5. **`semantic_differential.*` range.** `normalize_semantic_differentials()` produces 0–100 normalized values; the schema requires the original 1.0–7.0 bipolar scale (`minimum: 1.0, maximum: 7.0`).
6. **`gap_dimensions.*.score` can be `null` or a non-integer float.** Same integer/non-null requirement as gold-standard scores; `build_profile()` passes `gap_dimensions` through unmerged with raw float/`None` scores.
7. **`voice_stability_map` has an extra `per_dimension` key.** `compute_voice_stability()`'s full per-dimension breakdown is included alongside the two schema-required arrays (`stable_across_contexts`, `adapts_by_context`), but the schema has `additionalProperties: false`.
8. **`writing_sample_analysis` has no default.** `assemble_voice_profile(..., writing_sample_analysis=None)` defaults to `{}`, but the schema requires `sample_count` and `total_words` inside it — the caller must always compute and pass these explicitly, which isn't documented anywhere in the function's docstring.

## Impact

`bin/voice-scoring score` never calls `assemble_voice_profile()` — it only calls `build_profile()` (which has no schema in its own right) and writes that to `scores/self-report.json`. So this bug is invisible to the normal CLI pipeline. It only surfaces when something actually calls `assemble_voice_profile()` to produce the final `profile.json`, which per `agents/profile-synthesizer.md` is supposed to happen at the end of every interview. In practice, since the `profile-synthesizer` agent wasn't available in this session, the profile had to be assembled and schema-fixed by hand (see conversation this bug was filed from) — but if the agent *is* available and follows its own instructions literally, it would either fail schema validation and get stuck in a retry loop, or (more likely, since the instructions say "fix the output and retry" with no bound) improvise its own ad-hoc fixes each time, producing inconsistent profile shapes across sessions.

## Suggested approach

`assemble_voice_profile()` should own the full transform rather than delegating shape-matching to whatever calls it:

1. Always emit all 8 `gold_standard_dimensions` keys; for dimensions with `merged_score is None`, emit a `confidence: 0.0` placeholder (documented as "no data, module not administered for this branch") rather than omitting the key.
2. Round `score`/`self_report`/`observed` to `int` and fall back to the merged score itself when only one source is available, so the sub-object is always fully populated.
3. Compute a real `confidence` value (e.g. from `tier` and single- vs dual-source) instead of passing through `tier`/`weight_sr`/`weight_obs`.
4. Transform `calibrate()`'s raw `dimensions` map into `high_awareness_dimensions` / `blind_spots` / `aspirational_gaps` internally, and divide `overall_self_awareness` by 100.
5. Convert the 0–100 normalized semantic-differential scores back to the 1.0–7.0 scale before writing.
6. Round `gap_dimensions.*.score` to `int` and drop entries with `None` scores (or decide on a documented placeholder instead of dropping, for consistency with point 1).
7. Drop (or namespace separately, e.g. under `metadata`) the `per_dimension` stability breakdown before returning.
8. Compute `sample_count`/`total_words` from the session's `writing-samples/*.analysis.json` files internally instead of requiring the caller to pass them in.

A regression test that runs `assemble_voice_profile()` output through `jsonschema.validate()` against `voice-profile.schema.json` would catch all of the above and prevent drift between the two files in the future.

## Steps to Reproduce

```bash
# From a completed session with both self-report scores and NLP analysis:
.venv/bin/python -c "
import json, sys
sys.path.insert(0, 'scoring/src')
import jsonschema
from voice_scoring.profile_builder import assemble_voice_profile

scores = json.load(open('SESSION_DIR/scores/self-report.json'))
profile = assemble_voice_profile(
    build_result=scores['profile'],
    session_id='SESSION_ID',
    writer_type='academic_technical',
    identity_summary='...',
    calibration_report=scores.get('calibration'),
    semantic_differential=scores.get('semantic_differentials'),
)
schema = json.load(open('question-bank/schemas/voice-profile.schema.json'))
jsonschema.validate(profile, schema)  # raises ValidationError
"
```

## Priority

Medium. Doesn't block the CLI scoring pipeline (which bypasses `assemble_voice_profile()` entirely), but blocks the documented end-of-interview profile-synthesis step from working out of the box, and risks profiles with inconsistent shapes if agents each improvise their own fix.
