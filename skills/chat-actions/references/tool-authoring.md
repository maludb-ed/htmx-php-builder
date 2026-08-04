# Authoring the actions-server tools

How to write the tool descriptions and schemas for `{app}_actions_mcp`. The router's quality is a function of these descriptions — routing knowledge lives here, not in the system prompt.

## Anatomy shared by every tool

1. **Positive triggers** — the utterance shapes that should activate it ("go to…", "record/log…", "did N reps of…").
2. **Negative triggers** — what it is NOT for, naming the tool family that is ("Do NOT call this to answer questions — use the record/activity tools"). The router's real job is choosing *between* families.
3. **Typed schema with extraction guidance** — see the slot-filling rules below.
4. **Terminal semantics** — "After success: read back the interpretation in one short sentence and END THE TURN. No further tool calls."
5. **Actionable failure** — errors return closest candidates with descriptions so the model self-corrects in one retry or asks one short question.
6. **Structured result** — JSON `status` plus the machine payload the harness extracts (`navigate` directive, `undo_id`, `refresh` event name).

## `navigate` — routing is classification

```python
class NavigateInput(BaseModel):
    model_config = ConfigDict(extra="forbid")
    screen: str = Field(..., description="A screen id from the registry in this tool's description")
    params: dict[str, str] | None = Field(default=None,
        description="Optional prefill values; become query-string parameters on the destination")

@mcp.tool(name="navigate", annotations={"title": "Navigate", "readOnlyHint": True, "idempotentHint": True})
async def navigate(params: NavigateInput) -> str:
    """Take the user to a screen in the application.

    Call this when the user asks to go somewhere or see something: "go to…",
    "open…", "show me…", "take me to…", "back to the list". Do NOT call it to
    answer questions (use the record/activity tools) or to save data (use the
    action tools).

    `params` pre-fills the destination — {"name": "Bench Press"} on
    exercise-add pre-fills the create form's name field.

    After a success result: tell the user where they are in one short
    sentence and END THE TURN. At most one navigation per message.

    Screens:
      dashboard       — the home overview
      exercises-list  — browse or search exercises
      exercise-add    — create a new exercise (prefill: name, category)
      workout-log     — today's workout log
    """
```

- **Generate the screen list from the action manifest at server startup** — id + one-line "when the user wants…" phrase — and validate `screen` against the same manifest, so description and validation never drift.
- Unknown screen → return the closest matches with their descriptions.
- Success → `{"status": "success", "navigate": {"path": "/exercises/new?name=Bench+Press", "target": "#page-content"}}`.
- Past ~30–40 screens, split into `find_screen(query)` + `navigate(id)`; below that, one tool with the full list keeps routing to a single call.

## Action tools — extraction is the schema's job, not the tool's

The utterance-to-fields conversion ("Record 1 set of 12 reps bench press at 225lbs" → `reps=12, weight_lbs=225`) happens **inside the model** as it fills `tool_use` input. Write no parsing code; write the contract:

```python
class RecordSetInput(BaseModel):
    model_config = ConfigDict(extra="forbid")
    exercise: str = Field(..., description="Exercise name as the user said it, e.g. 'bench press'. Pass the name — the tool resolves it; never invent an id.")
    sets: int = Field(default=1, ge=1, le=20, description="Number of sets; 'a set' = 1")
    reps: int = Field(..., ge=1, le=200, description="Repetitions per set; dictation may spell numbers out ('twelve')")
    weight_lbs: float | None = Field(default=None, ge=0, description="Weight in POUNDS; convert kg × 2.205; omit for bodyweight")

@mcp.tool(name="workout_record_set", annotations={"title": "Record Set", "readOnlyHint": False, "destructiveHint": False})
async def workout_record_set(params: RecordSetInput) -> str:
    """Record completed set(s) in the workout log.

    Call when the user reports work they DID: "record/log 3 sets of bench at
    45", "did 12 reps of squats at 225". Executes immediately — no
    confirmation — and returns exactly what was recorded plus an undo handle.
    After success: read back the interpretation in one sentence and END THE TURN.
    """
```

Slot-filling rules that make dictated input reliable:

- **Unit in the field name** (`weight_lbs`), conversion rule in the description ("convert kg × 2.205").
- **Plausibility bounds** (`ge`/`le`) catch dictation mishears ("two twenty-five reps").
- **Names in, ids resolved server-side.** The tool fuzzy-matches "bench press" → `exercise_id` against the database; ambiguity fails usefully: *"Found 'Bench Press (Barbell)' and 'Bench Press (Dumbbell)' — which one?"* One call for the common case; one short question otherwise.
- **Screen context supplies defaults**: on an exercise's detail screen, "log 12 reps at 225" defaults `exercise` from the message's `data-record-id` context — note this in the description when it applies.

The tool body does only what the model can't: resolve entities, POST to the app's own endpoint with the action token, and return
`{"status": "success", "recorded": {"exercise": "Bench Press (Barbell)", "sets": 1, "reps": 12, "weight_lbs": 225}, "undo_id": "act_8231", "refresh": "workoutsChanged"}` —
the harness turns `refresh` into `HX-Trigger` and renders the Undo control from `undo_id`.

## navigate vs action tools — the 80/20

Identical: trigger structure, terminal act-and-end-turn semantics, fail-with-candidates errors, structured results. Different: `navigate` is **classification** against a closed registry and returns an `HX-Location` directive; action tools are **slot-filling** into a typed schema, resolve entities server-side, and return an undo handle plus an `HX-Trigger` event. Both are one tool call for the common case.
