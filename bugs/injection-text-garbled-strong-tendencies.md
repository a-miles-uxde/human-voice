# Bug: `format_profile_for_injection()` produces garbled "Strong tendencies" text

## Summary

`lib/profile.py::format_profile_for_injection()` — the function that writes `voice-prompt.txt`, the compact system-prompt injection text read by hooks/agents — tries to extract pole labels (e.g. "formal"/"casual") from `semantic_differential` dimension names by splitting on `_`. But `semantic_differential` in `profile.json` is keyed by composite dimension name (`formality`, `humor`, `audience_awareness`, ...), not by pole-pair (`formal_casual`). Splitting `"formality"` on `_` just returns `["formality"]`, so the output reads nonsense like:

```
Strong tendencies: strongly formality, strongly audience
```

instead of something like `strongly casual, strongly inclusive`.

## Environment

- human-voice (repo, `develop` branch, as of 2026-09-23)
- Reproduced on the real `~/.human-voice/voice-prompt.txt` generated for profile `william` (session `3cd77a4a-59c5-4550-8c0c-ce67f4600226`)

## Root Cause

**File**: `lib/profile.py`, lines 202–212

```python
sd = profile.get("semantic_differential", {})
if sd:
    extremes = []
    for pair, value in sd.items():
        if isinstance(value, (int, float)):
            if value <= 2.5:
                poles = pair.split("_")
                extremes.append(f"strongly {poles[0]}")
            elif value >= 5.5:
                poles = pair.split("_")
                extremes.append(f"strongly {poles[-1]}" if len(poles) > 1 else f"high {pair}")
```

The variable is named `pair` and the code assumes it looks like `"formal_casual"`, matching the raw per-item SD question IDs' pole labels (e.g. `SD-01: "Formal — Casual"`). But by the time a profile reaches this function, `semantic_differential` has already been through `normalize_semantic_differentials()` and `assemble_voice_profile()`, which key it by **composite dimension name** (see `question-bank/scoring/sd-dimension-mapping.json` and the `voice-profile.schema.json` `semantic_differential` property) — e.g. `formality`, `humor`, `complexity`, `authority`, `emotional_tone`, `narrativity`, `audience_awareness`, `personality`. None of these contain an underscore, so `pair.split("_")` is always a 1-element list and every branch collapses to `poles[0]` == the dimension name itself.

## A deeper issue, not just the split

Even fixing the split wouldn't fully solve this: `sd-dimension-mapping.json` shows several raw SD pairs feed into the *same* composite dimension —

- `complexity` ← SD-03 (Elaborate–Minimalist), SD-06 (Concrete–Abstract), SD-07 (Structured–Flowing), SD-13 (Precise–Evocative), SD-14 (Concise–Expansive)
- `authority` ← SD-04 (Assertive–Tentative), SD-08 (Direct–Diplomatic), SD-11 (Authoritative–Collaborative), SD-17 (Careful–Bold)
- `personality` ← SD-12, SD-13, SD-15, SD-16, SD-17, SD-20

So a composite `complexity` score of, say, 2.6 doesn't correspond to one specific pole pair — it's an average across five conceptually distinct bipolar scales. There is no single correct "strongly X" pole label for a composite dimension in general; the function's whole approach (treat the dimension name as a splittable pole-pair) only made sense for the raw per-item SD-01..SD-20 scores, not the merged dimension-level composites that `profile.json` actually stores.

## Impact

Every published profile's `voice-prompt.txt` — the file hooks and other agents read to inject voice guidance into an LLM system prompt — contains a "Strong tendencies" line that is either nonsensical (`strongly formality`) or silently misleading if a reader assumes it's a real pole label. Anything downstream that trusts this line (the observer protocol, other agents reading the compact injection) is working from garbled guidance.

## Suggested approach

Two options, not mutually exclusive:

1. **Minimal fix**: maintain a small `DIMENSION_POLE_LABELS` map (dimension name → representative `(low_pole, high_pole)` tuple, e.g. `"formality": ("casual", "formal")`, `"audience_awareness": ("inclusive", "exclusive")`) hand-picked from the dominant SD pair for each dimension, and use that instead of string-splitting. This at least produces grammatical, plausible-sounding output, though it papers over the many-to-one aggregation.
2. **More honest fix**: don't attempt pole-pair phrasing for composite dimensions at all. Report strong tendencies as `"low {dimension}"` / `"high {dimension}"` (the code already has this fallback for the `len(poles) > 1` false case — just always take that branch), or better, surface the *raw* per-item SD-01..SD-20 scores (which do have real, unambiguous pole pairs) for the "Strong tendencies" line instead of the merged per-dimension composites.

Option 2 is more defensible since it doesn't require maintaining a hand-picked representative-pole table that will drift out of sync with `sd-dimension-mapping.json` as dimensions gain or lose contributing SD items.

A unit test asserting `format_profile_for_injection()` never emits a "strongly {dimension_name}" string (i.e., the dimension name itself never leaks into the pole label) would catch regressions here.

## Steps to Reproduce

```bash
python3 -c "
from lib.profile import format_profile_for_injection
profile = {
    'identity_summary': 'test',
    'gold_standard_dimensions': {},
    'semantic_differential': {'formality': 2.33, 'audience_awareness': 1.67},
    'distinctive_features': [],
    'calibration': {},
}
print(format_profile_for_injection(profile))
"
# Strong tendencies: strongly formality, strongly audience
```

## Priority

Low-medium. Cosmetic/quality issue in a supplementary text field — doesn't block the interview or scoring pipelines, but degrades the one artifact meant to be directly useful outside the plugin (the compact voice-prompt injection).
