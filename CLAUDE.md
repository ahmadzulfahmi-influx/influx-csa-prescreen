# CLAUDE.md — pre-screening/csa

_Last updated: 2026-06-11 (v2.1 — reclassify categorical fields to filterable; recruiter filter view)_

## What this is

Standalone static pre-screen form for the Customer Support Agent (CSA) role at Influx.
Replaces the AI chat-based pre-screen (Prompt P / Prompt PE) that was part of the
`AI-assessment-demo` project. Decision rationale: feedback analysis showed the AI chat
was blind to operational-fit disqualifiers — see `legacy/README.md` for context.

## Architecture

Single static HTML file (`prescreen-form.html`) — no build step, no server required.
Open in any browser. Designed to match the production pre-screen form's visual layout.

```
Screen 1 (Intro) → Screen 2 (Form) → Screen 3a (Gate) or Screen 3b (Complete)
```

Gate result is shown in-page. On the complete screen, a **recruiter filter view** renders:
a candidate list (left) + filter panel (right) seeded with synthetic candidates, with the
just-submitted candidate pinned at the top. This demonstrates how `filterable{}` and
`prescreen_traits[]` drive recruiter filtering in the People app. The raw
`PrescreenResponses` JSON (the engineering contract) is available in a collapsible panel.

## Design System

Influx tokens (matching `AI-assessment-demo/index.html`):

| Token | Value | Use |
|---|---|---|
| `--tan` | `#FF4A00` | Primary, CTA buttons, active states |
| `--esp` | `#201B13` | Text |
| `--sand` | `#E9E5D6` | Page background |
| `--snow` | `#FFFFFF` | Card background |
| `--hazel` | `#D3C5BF` | Borders, dividers |
| `--char` | `#55524E` | Secondary text, hints |
| `--olive` | `#587322` | Success / clear state |

Font: Inter (Google Fonts). Icons: FontAwesome Duotone kit (same as demo).

## Gate Rules

Three **hard gates** (block progression):
1. `availability_not_full_time` — candidate can't work full-time (44 hrs/wk)
2. `english_below_bar` — self-rates English below `CONFIG.english_minimum`
3. `salary_expectation_above_range` — ask exceeds `CONFIG.salary_ceiling[currency]`

Everything else is **flag-and-continue** (`prescreen_status = "completed_flagged"`).

## Traits & Filterable Output

Every submission produces two extra blocks appended to the response JSON:

**`prescreen_traits: string[]`** — **boolean operational flags only** (v2.1). Tags for:
full-time availability, weekend/holiday/shift availability, voice-capable, has-competing-processes,
ic-accepted, returning-applicant. Categorical attributes have been removed — they live in
`filterable`. Trait slugs are a stable contract — do not rename without a migration.

**`filterable: {}`** — **structured recruiter-filter contract** (v2.1). Covers:
`country`, `city`, `education_level`, `field_of_study`, `experience_years`, `skills[]`,
`english_level`, `employment_status`, `notice_period`, `available_from`,
`salary_expectation` ({amount, currency}), `internet_mbps`.

Split rationale: mirrors the People app's filter architecture — categorical/scalar fields use
dedicated filters (`q[country_id_in]`, education select, etc.); boolean operational tags use the
generic Trait multi-select (`q[user_trait_positions_right_id_in]`).

See `prescreen-form-spec.md` → Trait Taxonomy, Filterable Block, and Recruiter Filter View
sections for the full tables and filter semantics.

## Config Block

Top of `prescreen-form.html` — `const CONFIG = { ... }`. HR replaces:
- `salary_range_display`: shown in the IC acknowledgment text (keep `{{salary_range}}`
  as placeholder in git — OD-5 from AI-assessment-demo applies here too)
- `salary_ceiling`: per-currency hard gate thresholds
- `english_minimum`: minimum accepted English level (`"basic"` = only gate Basic, most permissive)
- `required_shifts`: shift codes for flag check (e.g. `["regular","morning"]`)

## Required Fields (v2)

7 required fields — submission is blocked if any are blank:
1. Full-time availability
2. Earliest start date
3. English level
4. Education level
5. Years of relevant experience
6. Prior Influx application (Yes/No)
7. Salary expectation

Notice period, skills, field of study, voice, working days/times, location, specs are optional.

## Files

| File | Purpose |
|---|---|
| `prescreen-form.html` | The static form — open in browser to run (v2.1: click bug fixed, 6 new fields, reclassified traits/filterable, recruiter filter view) |
| `prescreen-form-spec.md` | Engineering spec v2.1: full TS interface, gate rules, boolean-only trait taxonomy, structured filterable block, recruiter filter semantics |
| `legacy/` | Original AI chat pre-screen prompt files (Prompt P, Prompt PE, template) + call notes (pre-screening-9jun2026.txt) |
| `.gitignore` | Excludes `*.mov` and `frames/` (may contain PII) |

## Running

```
# Open directly:
start prescreen-form.html

# Or serve locally:
python -m http.server 8080
# → http://localhost:8080/prescreen-form.html
```

## Extending / Integrating

See `prescreen-form-spec.md` for the full field contract. The engineering team at
`apps/people/` integrates by:
1. Replacing the Prompt P + Prompt PE pipeline with a static form submission
2. Parsing the extended `PrescreenResponses` JSON (incl. `prescreen_traits[]` + `filterable{}`)
3. Adding `knocked_out` to the `prescreen_status` enum
4. Indexing `prescreen_traits` for recruiter filtering; indexing `filterable` for range queries
5. Extending the "Chat Assessment Pre-Screening" admin block with new fields

## Reference

- `legacy/README.md` — why the AI pre-screen was replaced
- `AI-assessment-demo/context/architecture/AI-assessment-demo - Pre-Screen Pipeline.md`
- `AI-assessment-demo/planning_docs/AI_Assessment_Engineering_Spec_v1.1.md §12.8`
