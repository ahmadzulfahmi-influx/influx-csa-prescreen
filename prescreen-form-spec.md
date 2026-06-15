# Pre-Screen Form Spec — Customer Support Agent (CSA)

**Version:** 2.1
**Date:** 2026-06-11
**Status:** Draft for engineering review
**Replaces:** AI chat pre-screen via Prompt P / Prompt PE (see `legacy/`)

---

## Context

The AI behavioral chat is blind to operational-fit disqualifiers. Feedback analysis across 47
candidates (46.8% AI/RTL alignment; 0% alignment at the AI-Yes tier in the 2nd-interview cohort)
showed three categories the AI cannot assess:

1. **Availability** — Karlyn: part-time only (4 hrs/day; role needs 44 hrs/wk). AI scored Yes.
2. **Salary expectation** — candidates applying with a 2× ask; AI never surfaced it.
3. **English bar** — Raihan: AI interview showed stronger English than the live call revealed.

Additionally, a live 10-minute pre-screening call (2026-06-09) surfaced structured items that
are form-able but were not previously captured: start date, education, experience, skills,
and prior Influx application.

This spec defines a **static form** that collects these factors directly, before the AI chat.

---

## Pipeline Position

```
[Static Pre-Screen Form]
  ↓ candidate submits
Extended PrescreenResponses JSON  +  prescreen_traits[]  +  filterable{}
  ↓
Hard-gate evaluation (client-side / server-side Ruby)
  ├─ knocked_out → show gate screen; do NOT proceed to AI chat
  └─ clear / flagged → proceed to Prompt A (behavioral, unchanged)
                        prescreen_responses stored on CandidateSession
```

---

## Extended PrescreenResponses Interface

Extends the existing `PrescreenResponses` contract (canonical field names preserved).
New fields are additions only — no renames, no removals.

```typescript
interface ExtendedPrescreenResponses {

  // ── EXISTING FIELDS (preserved from Prompt PE contract) ─────────────────

  device_specs: {
    value:          string | null;   // free-text: "MacBook Pro M5 16GB"
    internet_speed: string | null;   // free-text or number: "100 Mbps"
    adequacy:       "adequate" | "inadequate" | "unclear" | null;
    evidence:       string | null;
  };

  competing_processes: {
    value:    "yes" | "no" | null;
    evidence: string | null;
  };

  shift_preference: {
    // Derived from working_days + working_times (see mapping below)
    ranking:  Array<"regular" | "morning" | "afternoon" | "graveyard" | "weekend"> | null;
    evidence: string | null;
  };

  voice_support: {
    comfortable: boolean | null;
    evidence:    string | null;
  };

  ic_status: {
    accepts:  boolean | null;
    evidence: string | null;
  };

  salary: {
    // Legacy field: candidate explicitly acknowledged the stated range
    acknowledged: boolean | null;
    evidence:     string | null;
  };

  // ── NEW FIELDS (feedback-driven additions) ──────────────────────────────

  availability: {
    full_time:  boolean | null;  // can commit 44 hrs/wk, 8 hrs/day
    start_date: string  | null;  // ISO 8601 date: "2026-07-15"
    evidence:   string  | null;
  };

  salary_expectation: {
    amount:   number | null;   // candidate's stated monthly expectation
    currency: string | null;   // ISO 4217: "IDR", "USD", "PHP", "MYR", …
    evidence: string | null;
  };

  english_self_rating: {
    level:    "basic" | "conversational" | "fluent" | "native" | null;
    evidence: string | null;
  };

  employment_status: {
    status:        "employed" | "employed_part" | "unemployed" | "student" | "freelancer" | null;
    notice_period: string | null;   // e.g. "1month" — populated if employed/employed_part
    evidence:      string | null;
  };

  education: {
    level:          "high_school" | "diploma" | "bachelors" | "masters" | "doctorate" | "other" | null;
    field_of_study: string | null;   // optional free-text
    evidence:       string | null;
  };

  experience: {
    years:    number | null;   // 0 or positive; banded into traits client-side
    evidence: string | null;
  };

  skills: {
    selected: string[];         // curated slugs (see Curated Skill Slugs below)
    other:    string | null;    // free-text; stored for reference, NOT a trait
    evidence: string | null;
  };

  prior_application: {
    applied_before: boolean | null;
    evidence:       string | null;
  };

  job_search_intent: {
    why_csa:  string | null;   // candidate's stated motivation (free-text)
    evidence: string | null;
  };

  // ── TRAIT + FILTERABLE BLOCKS (appended at submit) ───────────────────────

  // Boolean operational flags only — see Trait Taxonomy.
  // Categorical/scalar attributes (english, status, education, experience,
  // skills, country) live in filterable{} — matching how the People app
  // separates dedicated filters (q[country_id_in]) from the generic Trait
  // multi-select (q[user_trait_positions_right_id_in]).
  prescreen_traits: string[];

  filterable: {
    country:            string | null;                           // e.g. "Indonesia"
    city:               string | null;
    education_level:    string | null;                           // slug, e.g. "bachelors"
    field_of_study:     string | null;
    experience_years:   number | null;
    skills:             string[];                                // curated slugs array
    english_level:      string | null;                           // e.g. "fluent"
    employment_status:  string | null;                           // e.g. "employed"
    notice_period:      string | null;
    available_from:     string | null;                           // ISO date
    salary_expectation: { amount: number; currency: string } | null;
    internet_mbps:      number | null;
  };

  // ── FORM METADATA ───────────────────────────────────────────────────────

  form_meta: {
    candidate_name:  string;
    candidate_email: string;
    working_days:    string[];  // raw: ["weekdays", "weekends", "public_holidays"]
    working_times:   string[];  // raw: ["regular", "morning", "afternoon", "night"]
    country:         string | null;
    city:            string | null;
    submitted_at:    string;    // ISO 8601
  };
}
```

---

### `shift_preference.ranking` derivation

The form captures days and times as multi-select checkboxes. The server/client maps these to
the legacy `shift_preference.ranking` array for downstream compatibility:

| Time selected | Days include weekend? | Mapped shift |
|---|---|---|
| regular (9–6 PM) + weekdays | no | `regular` |
| morning (6–3 AM) | any | `morning` |
| afternoon (12–9 PM) | any | `afternoon` |
| night (8 PM–5 AM) | any | `graveyard` |
| any time | weekends | `weekend` |

Order in `ranking[]` = order the candidate selected them (first selected = most preferred).

---

## Gate Rules

### Hard Gates (do NOT proceed to AI chat)

| Gate key | Condition | `prescreen_flagged_reasons` entry |
|---|---|---|
| `availability_not_full_time` | `availability.full_time == false` | `"availability_not_full_time"` |
| `salary_expectation_above_range` | `salary_expectation.amount > CONFIG.salary_ceiling[currency]` | `"salary_expectation_above_range"` |
| `english_below_bar` | `english_self_rating.level` index < `CONFIG.english_minimum` index in `[basic, conversational, fluent, native]` | `"english_below_bar"` |

When any hard gate fires:
- Set `prescreen_status = "knocked_out"` *(new enum value — see schema delta below)*
- Store `prescreen_flagged_reasons` (may contain multiple)
- Show polite gate screen; do NOT link/proceed to assessment URL
- `prescreen_traits` and `filterable` are still populated on all submissions (server-side consumption)

**Note on English gate:** Self-rating is a proxy — the Raihan case shows candidates
over-rate their English. The gate applies only at "Basic" (the lowest band), so the bar is
intentionally low. Flag-and-continue is still recorded for "Conversational" if the role
config requires "Fluent". See `CONFIG.english_minimum` below.

### Flag-and-Continue (proceed, but tag the session)

| Flag key | Condition |
|---|---|
| `voice_support_declined` | `voice_support.comfortable == false` AND `CONFIG.voice_required == true` |
| `ic_status_declined` | `ic_status.accepts == false` AND `CONFIG.ic_is_knockout == true` |
| `has_competing_processes` | `competing_processes.value == "yes"` |
| `shift_preference_mismatch` | top-2 of `shift_preference.ranking` have zero overlap with `CONFIG.required_shifts` |
| `device_specs_unclear` | `device_specs.adequacy == "unclear"` — **server-side only**; client form cannot reliably evaluate free-text specs. Evaluated against `device_adequacy_criteria` template config. |
| `salary_acknowledged_no` | `salary.acknowledged == false` |

---

## `prescreen_status` Enum Delta

Add `"knocked_out"` to the existing enum:

```ruby
# Current (apps/people/):
enum prescreen_status: {
  not_started:        "not_started",
  in_progress:        "in_progress",
  completed_clear:    "completed_clear",
  completed_flagged:  "completed_flagged"
}

# Proposed addition:
  knocked_out:        "knocked_out"
```

Migration: no default change needed for existing rows (retroactive status unchanged).

---

## Trait Taxonomy

`prescreen_traits` is a flat array of **boolean operational flags** — affirmative-only
kebab-case tags. A tag is emitted only when the answer is affirmative/selected.

**Important — reclassification (v2.1):** Categorical and scalar attributes (country,
education level, experience, skills, English level, employment status) have been moved out of
traits and into `filterable{}` as structured fields. This mirrors how the People app works:
those attributes use dedicated filters (`q[country_id_in]`, education select, etc.), while the
generic Trait multi-select (`q[user_trait_positions_right_id_in]`) holds boolean operational
tags. Traits removed: `english-*`, `status-*`, `education-*`, `experience-*`, `skill-*`,
`country-*`. Server-side: drop/ignore those slugs from any existing index.

Tags are a stable contract — **do not rename the remaining slugs without a migration**.

| Source field | Condition | Trait tag |
|---|---|---|
| `availability.full_time` | `true` | `full-time-available` |
| `form_meta.working_days` | includes `weekends` | `available-weekends` |
| `form_meta.working_days` | includes `public_holidays` | `available-public-holidays` |
| `form_meta.working_times` | includes `regular` | `shift-regular` |
| `form_meta.working_times` | includes `morning` | `shift-morning` |
| `form_meta.working_times` | includes `afternoon` | `shift-afternoon` |
| `form_meta.working_times` | includes `night` | `shift-night` |
| `voice_support.comfortable` | `true` | `voice-capable` |
| `competing_processes.value` | `"yes"` | `has-competing-processes` |
| `ic_status.accepts` | `true` | `ic-accepted` |
| `prior_application.applied_before` | `true` | `returning-applicant` |

**Note:** All skill-related filtering uses `filterable.skills[]` (the curated slug array), not
trait tags. `skills.other` free-text is never indexed.

### Curated Skill Slugs

The client form exposes exactly these slugs. Server-side indexing should mirror this set:

| Slug | Label |
|---|---|
| `email-support` | Email support |
| `live-chat` | Live chat |
| `phone-voice` | Phone / voice |
| `crm-tools` | CRM tools (e.g. Zendesk, Freshdesk, Intercom) |
| `ticketing` | Ticketing systems |
| `troubleshooting` | Troubleshooting |
| `technical-support` | Technical support |
| `multilingual` | Multilingual |
| `data-entry` | Data entry |
| `social-media` | Social media support |
| `sales-upsell` | Sales / upsell |
| `complaint-handling` | Complaint handling |

---

## Filterable Block

`filterable` is the **structured recruiter-filter contract** — intentional denormalization of
categorical/scalar values for efficient range/value/multi-select queries. Index all fields.

Use `filterable` for: country/education/skills/english/status lookups and range queries.
Use `prescreen_traits` for: boolean presence/absence checks (available weekends, voice capable, etc.).

| Field | Type | Source |
|---|---|---|
| `country` | string or null | `form_meta.country` |
| `city` | string or null | `form_meta.city` |
| `education_level` | slug or null | `education.level` (e.g. `"bachelors"`) |
| `field_of_study` | string or null | `education.field_of_study` |
| `experience_years` | number or null | `experience.years` |
| `skills` | string[] | `skills.selected` — curated slugs only |
| `english_level` | slug or null | `english_self_rating.level` (e.g. `"fluent"`) |
| `employment_status` | slug or null | `employment_status.status` |
| `notice_period` | string or null | `employment_status.notice_period` |
| `available_from` | ISO date or null | `availability.start_date` |
| `salary_expectation` | `{ amount, currency }` or null | `salary_expectation.{amount,currency}` |
| `internet_mbps` | number or null | parsed from `device_specs.internet_speed` |

### Recruiter Filter View

The demo's complete screen (post-submission) renders a functional recruiter filter view
modelled on the People app layout: candidate list (left) + filter panel (right) + live count.
Seed candidates are shown alongside the just-submitted one to demonstrate the filter.

Filter semantics implemented client-side and documented here for the server-side mirror:
- **Country / Education / Skills / Traits** — checkbox multi-select; OR within a group, AND across groups
- **English level** — "at least X" ordered filter (basic < conversational < fluent < native)
- **Employment status** — exact match
- **Experience years** — numeric min/max range
- **Salary max** — numeric upper bound, compared within the same currency only (no FX conversion; rows in a different currency are never falsely excluded)

---

## Template Config Fields

Reuses the existing `chat_assessment_template` config keys plus new additions:

| Field | Type | Purpose |
|---|---|---|
| `prescreen_enabled` | boolean | Master toggle |
| `device_specs.enabled` | boolean | Show device/internet fields |
| `device_specs.is_knockout` | boolean | Flag if inadequate |
| `device_adequacy_criteria` | string | e.g. "Min 8GB RAM, 25 Mbps download" |
| `competing_processes.enabled` | boolean | Show competing-processes question |
| `shift_preference.enabled` | boolean | Show working days/times fields |
| `shift_preference.is_knockout` | boolean | Flag on shift mismatch |
| `shift_preference.required_shifts` | string[] | e.g. `["regular","morning"]` |
| `voice_support.enabled` | boolean | Show voice question |
| `voice_support.is_knockout` | boolean | Flag if No |
| `ic_status.enabled` | boolean | Show IC acknowledgment |
| `ic_status.is_knockout` | boolean | Usually true |
| `salary.enabled` | boolean | Show salary range acknowledgment |
| `salary.is_knockout` | boolean | Flag if not acknowledged |
| `salary_range` | string | Injected into `{{salary_range}}` display |
| *(new)* `availability.enabled` | boolean | Show full-time availability question |
| *(new)* `availability.is_hard_gate` | boolean | Default: true |
| *(new)* `salary_expectation.enabled` | boolean | Show salary expectation input |
| *(new)* `salary_expectation.ceiling_usd` | number | Hard gate ceiling in USD equivalent |
| *(new)* `salary_expectation.ceiling_idr` | number | Hard gate ceiling in IDR |
| *(new)* `english.enabled` | boolean | Show English self-rating |
| *(new)* `english.minimum_level` | string | `"basic"` \| `"conversational"` \| `"fluent"` — gate if below |
| *(new)* `employment_status.enabled` | boolean | Show employment status + notice period |
| *(new)* `education.enabled` | boolean | Show education fields |
| *(new)* `experience.enabled` | boolean | Show experience field |
| *(new)* `skills.enabled` | boolean | Show skills multi-select |
| *(new)* `prior_application.enabled` | boolean | Show prior-application question |
| *(new)* `job_search_intent.enabled` | boolean | Show "why this role" question |

---

## Form Fields Reference

| # | Label | Control | Key | Required | Gate / Flag |
|---|---|---|---|---|---|
| 1 | Can you commit to full-time work? | Radio: Yes / No | `availability.full_time` | ✓ | **HARD GATE** if No |
| 2 | Which days do you prefer working? | Checkboxes: Weekdays, Weekends, Public holidays | `form_meta.working_days` | — | — |
| 3 | Which times do you prefer working? | Checkboxes: Regular / Morning / Afternoon / Night | `form_meta.working_times` → `shift_preference.ranking` | — | Flag on mismatch |
| 4 | What is the earliest date you can start? | Date picker | `availability.start_date` | ✓ | — |
| 5 | Are you comfortable providing voice support? | Radio: Yes / No | `voice_support.comfortable` | — | Flag if No |
| 6 | How would you rate your English? | Select: Basic / Conversational / Fluent / Native | `english_self_rating.level` | ✓ | **HARD GATE** if below minimum |
| 7 | Current employment status | Select: Employed / Employed (part-time) / Unemployed / Student / Freelancer | `employment_status.status` | — | — |
| 8 | Notice period | Select: Immediate / <2 weeks / 2–4 weeks / 1 month / >1 month (conditional: shown only when employed) | `employment_status.notice_period` | — | — |
| 9 | Highest level of education | Select: High school / Diploma / Bachelor's / Master's / Doctorate / Other | `education.level` | ✓ | — |
| 10 | Field of study | Text | `education.field_of_study` | — | — |
| 11 | Years of relevant experience | Number (≥0) | `experience.years` | ✓ | — |
| 12 | Relevant skills | Checkboxes (12 curated slugs) + free-text Other | `skills.{selected,other}` | — | — |
| 13 | Have you applied to Influx before? | Radio: Yes / No | `prior_application.applied_before` | ✓ | — |
| 14 | What draws you to a Customer Support role? | Textarea (optional) | `job_search_intent.why_csa` | — | — |
| 15 | Any active recruitment processes? | Radio: Yes / No | `competing_processes.value` | — | Flag if Yes |
| 16 | Salary expectation (per month) | Number + currency select | `salary_expectation.{amount,currency}` | ✓ | **HARD GATE** if above ceiling |
| 17 | IC arrangement acknowledgment | Checkbox | `ic_status.accepts` | — | Flag if unchecked |
| 18 | Country | Select | `form_meta.country` | — | — |
| 19 | City | Text | `form_meta.city` | — | — |
| 20 | Computer specifications | Textarea | `device_specs.value` | — | Flag if inadequate (server-side) |
| 21 | Internet speed (Mbps) | Number | `device_specs.internet_speed` | — | Flag if below criteria (server-side) |

**Required fields (7):** full-time availability (#1), earliest start date (#4), English level (#6),
education (#9), years of experience (#11), prior application (#13), salary expectation (#16).

---

## Pipeline Continuity Note

The static form **replaces** Prompt P + Prompt PE in the pre-screen phase. The downstream
pipeline is unchanged:

```
[Static Form] → ExtendedPrescreenResponses JSON
                ↓
                Hard-gate eval (Ruby — same logic, extended for new fields)
                ↓
                prescreen_status + prescreen_flagged_reasons + prescreen_traits + filterable
                stored on CandidateSession
                ↓ (if not knocked_out)
                Prompt A (Behavioral Interviewer) — unchanged
                ↓
                Prompt B → Prompt C → AssessmentReport → DB → recruiter view
```

The recruiter's "Chat Assessment Pre-Screening" block in the People App admin should be
extended to display the new fields (availability, salary_expectation, english_self_rating,
employment_status, education, experience, skills, prior_application) and support filtering
by `prescreen_traits` and `filterable` values.

---

## Demo File

→ [prescreen-form.html](prescreen-form.html) — standalone static implementation of this spec.
   Influx design system, vanilla JS, client-side gate evaluation, JSON output panel.
   Open in browser, no server required.

**Configuration block** is at the top of `prescreen-form.html` — `const CONFIG = { ... }`.
Replace `{{salary_range}}` with the actual range before deploying.
