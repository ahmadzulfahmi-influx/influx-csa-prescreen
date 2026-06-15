# Chat Assessment — Pre-Screen Template Prompt

This is the HR-editable portion of Prompt P (the pre-screen interviewer). It defines
**which items to ask** and the wording guidance for each. The persona, conversation
mechanics, guardrails, opening, transitions, and closing are hardcoded in
`ChatAssessment::Prescreener::SYSTEM_PROMPT_BASE` — do not duplicate them here.

Paste the content below into `chat_assessment_template.prescreen_prompt` when creating
or adapting a template. Toggle items per role using the template config:
each item has `enabled` and `is_knockout` flags in the DB. The application
compiles this template at runtime to include only `enabled` items — the LLM
sees only what it should ask.

---

**OD-5 note:** The `{{salary_range}}` placeholder below must remain a placeholder in the
committed file. Actual figures live in the `chat_assessment_template.salary_range`
DB field and are injected at runtime. Never commit actual salary numbers here.

---

```markdown
## PRESCREEN CHECKLIST

Work through these items in order. For each item, follow the wording guidance but
adapt phrasing to fit the natural conversation. One clarifying follow-up per item maximum.

---

### ITEM 1: Device Specifications & Internet Speed

Collect the candidate's device setup and internet speed. Both are needed.

Wording guidance:
"First up — could you describe your device setup for us? Things like whether you're on a
laptop or desktop, and your specs if you know them. Also, any idea how fast your internet
connection is? An approximation is totally fine."

If they describe the device but not internet speed:
"And do you have a rough idea of your internet speed? Even just whether it's fast, slow,
or somewhere in between is helpful."

Note: Do not share adequacy criteria with the candidate. Do not use the words "minimum"
or "requirement."

---

### ITEM 2: Other Recruitment Processes

Find out if the candidate has any active applications or offers elsewhere. This is
informational — no expected answer.

Wording guidance:
"Are you currently in any other recruitment processes right now? No need to go into
detail — it's just useful context for us."

---

### ITEM 3: Working Hours & Shift Preference

Walk the candidate through the working arrangement, then ask them to **rank** all shifts from most to least preferred. Ranking captures backup options — useful when scheduling can't always match the first choice.

Information to share:
- 44 hours per week, 8 hours per day
- Regular schedule: Monday to Friday; once every two weeks includes a weekend day
- Shift options (numbered for ranking):
  1. Regular — 9 AM to 5 PM
  2. Morning — starts at 5 or 6 AM
  3. Afternoon — noon to 9 PM
  4. Graveyard — 8 PM to 4 AM
  5. Weekend — Wednesday to Sunday

Wording guidance:
"Let me walk you through how hours work here. It's 44 hours a week, 8 hours a day.
The standard schedule is Monday to Friday — though once every two weeks there's a
weekend day included. Here are the shift options:

1. Regular — 9 AM to 5 PM
2. Morning — starts at 5 or 6 AM
3. Afternoon — noon to 9 PM
4. Graveyard — 8 PM to 4 AM
5. Weekend — Wednesday to Sunday

Could you rank these from most to least preferred? Just type the numbers in order —
for example: 1, 3, 2, 5, 4"

Note: Accept any ranking format the candidate uses (names, numbers, or a mix). Prompt PE will normalize. If a candidate says "1 and 3 are fine, the rest I'd rather not," that's a valid partial ranking — accept it and move on.

---

### ITEM 4: Voice Support Comfort

[Role-conditional. Enable only for client-facing roles via `voice_support.enabled = true`
in the template config. If not enabled, this item is omitted entirely.]

Ask if the candidate is comfortable providing voice support (phone) as part of the role.

Wording guidance:
"For this role, there are times when customers prefer to speak by phone rather than
via chat. Are you comfortable with voice support being part of your day-to-day work?"

---

### ITEM 5: Independent Contractor Status

Explain the IC arrangement and confirm the candidate is comfortable with it.

Information to share:
- This role is an independent contractor position
- No employer-provided health insurance
- No employer-side tax withholding or coverage

Wording guidance:
"One practical thing to cover — this role is structured as an independent contractor
position. That means there's no employer-provided health insurance or tax coverage.
Just want to make sure you're aware of that arrangement and are comfortable with it."

If they seem uncertain or ask for clarification:
"That's the current structure for this role — I'd recommend confirming the specifics
with the team at the next stage. For now, just helpful to know if you're generally
comfortable with that kind of setup."

---

### ITEM 6: Salary Range

Share the starting salary range and confirm the candidate is aware.

Wording guidance:
"Last thing — the starting salary for this role is {{salary_range}}. That's the
entry-level range; there'll be more to discuss on specifics at the next stage. Just
wanted to make sure that's on your radar and you're comfortable continuing on that basis."
```

---

## What's hardcoded (do NOT include in the template)

Already in `ChatAssessment::Prescreener::SYSTEM_PROMPT_BASE`:

- YOUR IDENTITY
- CONVERSATION FLOW mechanics
- OPENING greeting
- NEUTRAL PRESENTATION rules
- TRANSITIONS wording
- GUARDRAILS (G1–G5)
- CLOSING + `[PRESCREEN_COMPLETE]` sentinel

## Template config fields (set in admin panel, not here)

| Field | Type | Purpose |
|---|---|---|
| `prescreen_enabled` | boolean | Master toggle for the entire pre-screen phase |
| `device_specs.enabled` | boolean | Ask Item 1 |
| `device_specs.is_knockout` | boolean | Flag if device/internet inadequate |
| `device_adequacy_criteria` | string | Criteria passed to Prompt PE (e.g. "Min 8GB RAM, 10 Mbps download") |
| `competing_processes.enabled` | boolean | Ask Item 2 |
| `competing_processes.is_knockout` | boolean | Rarely set — informational by default |
| `shift_preference.enabled` | boolean | Ask Item 3 |
| `shift_preference.is_knockout` | boolean | Set if role has strict shift requirement |
| `shift_preference.required_shifts` | string[] | If is_knockout, flag if none of the candidate's top-N ranked choices (default: top 2) overlap with this list |
| `voice_support.enabled` | boolean | Ask Item 4 (client-facing roles only) |
| `voice_support.is_knockout` | boolean | Flag if candidate is not comfortable |
| `ic_status.enabled` | boolean | Ask Item 5 |
| `ic_status.is_knockout` | boolean | Usually true — IC arrangement is non-negotiable |
| `salary.enabled` | boolean | Ask Item 6 |
| `salary.is_knockout` | boolean | Flag if candidate explicitly declines the range |
| `salary_range` | string | Injected into `{{salary_range}}` at runtime |
