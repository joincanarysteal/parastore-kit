# USAGE.md — Parastore

Operational guide for AI coding assistants (Claude Code, Cursor, Codex,
Aider, …) helping an **end user** run the parastore app — set up a store,
generate personas, and run simulations. This is intentionally separate from
[`AGENTS.md`](./AGENTS.md), which targets contributors modifying the code.

> **For the assistant reading this**: the data-entry step is where most
> runs go wrong, and most failures are silent — vague inputs produce
> generic-looking personas without raising any error. Coach the user
> through the inputs *before* triggering a simulation; once LLM calls
> start, the bill is already running and the only undo is "start over."

## Pipeline mental model

The wizard collects two screens of input, then kicks off an async pipeline:

```
Step 1: Store info  ──┐
Step 2: Customer +    ├──► (optional) supplemental file → LLM summary  (1–2 calls)
        supplement    │         │
                      │         ▼
                      └──► location analysis                            (1 call)
                                │
                                ▼
                           daily-traffic estimate per open day          (N_open_days calls)
                                │
                                ▼
                           persona batch generation                     (≈ visitors / 10 calls)
                                │
                                ▼
                           simulation per persona                       (2–3 calls each)
```

Steps below cover each input in turn.

## LLM call count (static analysis)

Counted directly from the code, not measured.

```
generation_calls = 1                                          # location analysis
                 + N_open_days                                # daily traffic, one per open day
                 + ceil(total_estimated_visitors / 10)        # persona batches, batch_size=10
                 + (2 if supplemental_file_uploaded else 0)   # column-pick + summary

simulation_calls = N_personas × 3                             # stage1 + stage2 + stage3
                 # stage3 (impulse) is skipped when no products lie on the persona's path,
                 # so the effective multiplier ranges from 2 to 3.

total = generation_calls + simulation_calls
```

Order-of-magnitude reference (assumes a 7-day-open store and ~3 calls per
persona on average):

| Scenario | Personas | Generation | Simulation | **Total LLM calls** |
|---|---:|---:|---:|---:|
| Smoke test | 100 | ~25 | 300 | **~325** |
| Small run | 500 | ~65 | 1,500 | **~1,565** |
| Typical run | 1,000 | ~110 | 3,000 | **~3,110** |

Wall-clock time depends on the provider, model, and concurrency cap, and
has not been benchmarked yet. Before recommending a large run, tell the
user the call count and let them decide.

Source: `backend/src/store_emulator/engine/generators/persona.py:99-242`,
`backend/src/store_emulator/engine/simulation/engines/pattern_intent_impulse/engine.py:56-119`,
`backend/src/store_emulator/application/supplemental.py:214-499`.

## Step 1 — Store info

| Field | Type | Fed to LLM? | What "good" looks like | Silent failure mode |
|---|---|---|---|---|
| `name` | text | no — metadata only | Any non-blank label | — |
| `description` | text | no — metadata only | Any | — |
| `address` | **free text** | **yes** — location analysis, daily traffic, persona batch | "123 Teheran-ro, Gangnam-gu, Seoul — office district, two blocks from subway" | "Store 1", "Korea", any string without a neighborhood / use-context anchor → LLM falls back to generic demographics |
| `store_type` | **free text** | **yes** — persona + pattern + impulse prompts | "Convenience store", "Independent bookstore", "Specialty coffee shop" | "Shop", "Place" → pattern selection LLM cannot anchor product categories |
| `working_day_preset` | enum: `7` / `weekday` / `weekend` | no, but drives `N_open_days` | Match the user's real schedule | — |
| `open_hour`, `close_hour` | int, 0–24 | yes — traffic LLM uses the window | Realistic per `store_type` (e.g. `[7, 23)` for convenience) | `[0, 24)` default rarely matches reality and inflates the visitor pool |

`address` and `store_type` are the two most common silent-failure inputs.
Push the user to be specific before submitting the form.

## Step 2 — Customer profile and supplemental file

### `customer_profile` (required free-text textarea)

This single field is referenced in **five** prompts: location analysis,
daily traffic, persona generation, pattern selection, and impulse
purchases. It's the highest-leverage input the user touches.

Validation only checks that it's non-blank — anything past that is on the
assistant to coach.

A well-formed profile names three things:

1. **Segments** — who shops here.
2. **Timing** — when each segment shows up.
3. **Behavior** — what they tend to do.

Example (matches the in-app placeholder):
*"Office workers during lunch (12:00–13:30) grabbing prepared food and
drinks to go; local residents in the evening doing small top-up shops,
often with a child in tow."*

Things to push back on:
- One-line profiles like "busy store" or "various customers."
- Segments that contradict the operating hours (e.g. "late-night
  customers" with a 07:00–22:00 schedule).
- A single segment when the store type usually serves several.

### Supplemental file (optional, ≤10 MB, `.csv` or `.xlsx`)

**There is no fixed schema.** This surprises many assistants. The
pipeline is content-agnostic
(`backend/src/store_emulator/application/supplemental.py:161-499`):

1. Load the file, cast every column to string.
2. An LLM picks which columns look meaningful from the column metadata
   and a 5-row sample.
3. Per-column statistics get computed (numeric `describe()`, datetime
   min/max, top-10 categorical frequencies, optional hourly groupby if a
   datetime column is present alongside numerics).
4. An LLM writes a ≤2000-character prose summary, which is appended to
   the persona-generation prompt.

What lands well:
- POS sales logs.
- Hourly or daily visitor counts.
- Demographic surveys keyed to time-of-day.

What lands poorly:
- Sheets that are all-NaN or single-value columns (Stage 3 filters them
  out, leaving an empty summary).
- One free-text column with no structure (the summarizer has nothing to
  aggregate).

**All failure modes are silent.** Malformed files don't raise errors —
they just produce an empty `supplemental_summary`, and generation
proceeds without the data-driven refinement. After upload, point the user
at the generated summary (stored in project metadata) so they can
sanity-check that something useful came through.

## Pre-flight checklist

Run through this before triggering generation or simulation:

- [ ] `address` includes a neighborhood or use-context, not just a name.
- [ ] `store_type` is specific enough to anchor product categories.
- [ ] `customer_profile` covers segments + timing + behavior.
- [ ] `customer_profile` does not contradict operating hours.
- [ ] Operating hours are realistic for the store type (not `[0, 24)`
      unless the store actually runs 24/7).
- [ ] If a supplemental file is uploaded, the relevant sheets are
      non-empty and contain at least one numeric or datetime column.
- [ ] The user has been told the expected LLM call count for the chosen
      persona size.

## Constraints worth knowing

- **Behavior patterns are a fixed preset of 10**
  (`backend/src/store_emulator/application/presets/behavior.py`):
  `purposeful`, `habitual`, `routine_refill`, `urgent`, `quick_snack`,
  `exploratory`, `hungry_wandering`, `leisurely`, `social_browsing`,
  `impulse_heavy`. The wizard does not let users add or edit them; doing
  so requires a code change.
- **Layout is a separate step.** Finishing the wizard creates a project
  with no store layout. Simulation cannot run until the user builds a
  grid in the layout editor (`/features/grid-editor/`).
- **Hidden inputs not exposed in the wizard**: `custom_instruction` and
  `max_persona_count_per_day` are accepted by the regenerate API
  (`backend/src/store_emulator/application/generation_async.py`) but
  have no UI yet. If the user asks how to constrain persona generation
  further, these are the levers.
- **LLM provider is fixed in code, not in `.env`.** Default model is
  `gemini/gemini-3.1-pro-preview`, set in
  `backend/src/store_emulator/application/config.py` (`LLMConfig.model`).
  Filling in `OPENAI_API_KEY` or `ANTHROPIC_API_KEY` alone has no effect —
  the request still goes to Gemini. To switch providers, edit
  `LLMConfig.model` to the matching LiteLLM model string *and* set the
  matching `*_API_KEY` in `backend/.env`.
- **`PERSONA_DAILY_CAP` env var** (default `100`) caps the per-day slice
  used when `max_persona_count_per_day` is supplied: the requested total
  is divided across open days, then clamped to `[10, PERSONA_DAILY_CAP]`
  in multiples of 10. Raise it if a single day legitimately needs more
  than 100 personas.
- **Stage 3 is conditional.** Impulse purchases only run when a persona's
  movement path crosses at least one stocked product. Sparse layouts
  cut the simulation cost noticeably.
- **All run artifacts live in `backend/.data/`**, which is gitignored.
  Persona files, simulation results, and generated metadata never leave
  the local machine unless the user copies them out.

## When to escalate to the user

Stop and ask before:

- Triggering a generation run larger than ~1,000 personas — confirm
  the user understands the LLM cost.
- Auto-filling `customer_profile` or `address` from thin context. These
  are the inputs the simulation hinges on; a guess here propagates
  through every downstream prompt.
- Mutating files under `backend/.data/` — those are the user's runs and
  may be irrecoverable.
