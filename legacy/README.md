# Legacy: AI Chat-based Pre-Screen Prompts

These three prompt files were the original AI-driven pre-screen implementation for the
Customer Support Agent (CSA) role. They powered an LLM conversation (via Prompt P) that
asked candidates logistics and eligibility questions before the behavioral interview.

## Why they're here

Feedback analysis (`AI-assessment-demo/planning_docs/feedbacks/markdown/`) showed that
the chat-based pre-screen had a critical blind spot: **operational-fit disqualifiers
(availability, salary expectation above range, English bar) were invisible to behavioral
chat**. The clearest examples:

- **Karlyn** — AI gave Yes; 2nd-interview ended in minutes because she was part-time only
  (4 hrs/day; role needs 44 hrs/wk). The availability field was on the application — AI
  never asked.
- **Callie / Indriyani-type** — salary expectation ~2× the offered range; never surfaced.
- **Raihan** — weaker live English than the AI transcript showed; career intent mismatch.

The decision (2026-06-11): **replace the AI chat pre-screen with a static form** that captures
these fields directly, with hybrid hard-gating for the three critical disqualifiers.

## What replaced them

→ [../prescreen-form.html](../prescreen-form.html) — standalone static form (Influx design system)  
→ [../prescreen-form-spec.md](../prescreen-form-spec.md) — engineering spec (field contract,
  gate rules, extended `PrescreenResponses` schema, `prescreen_status` enum delta)

## Files in this folder

| File | What it was |
|---|---|
| `chat-assessment-prescreen-system-prompt-v1.2.md` | Hardcoded Prompt P base — persona, mechanics, guardrails G1–G5, `[PRESCREEN_COMPLETE]` sentinel |
| `chat-assessment-prescreen-template-prompt-v1.2.md` | HR-editable checklist (6 items: device, competing processes, shift preference, voice, IC status, salary range placeholder) |
| `chat-assessment-prescreen-extractor-prompt-v1.2.md` | Hardcoded Prompt PE — converts pre-screen transcript → `PrescreenResponses` JSON |

## Original pipeline reference

```
Prompt P (chat interviewer) → transcript → Prompt PE (extractor) → PrescreenResponses JSON
  → Ruby knockout eval → prescreen_status + prescreen_flagged_reasons → proceed to Prompt A
```

Architecture docs:
- `AI-assessment-demo/context/architecture/AI-assessment-demo - Pre-Screen Pipeline.md`
- `AI-assessment-demo/context/decisions/2026-05-26 - AI-assessment-demo pre-screen phase approach.md`
- `AI-assessment-demo/planning_docs/AI_Assessment_Engineering_Spec_v1.1.md §12.8`
