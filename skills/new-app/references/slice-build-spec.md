# Slice build spec — template

One spec per Phase 3 vertical slice, written by the planning-class model at the end of Phase 1 and approved with the schema. The spec's job: a worker model building this slice makes **substitutions, never decisions**. Store specs under `docs/build-specs/{entity}.md` in the application repo.

The worker's standing instruction: *build this slice exactly like the exemplar slice, applying this spec. If anything here is ambiguous, conflicting, or missing, STOP — record the exact question under Open Questions and escalate. Do not improvise.*

---

```markdown
# Build spec: {entity} slice

Exemplar to replicate: {exemplar-entity} (built by the planning-class model)
Schema tables: {table names from the approved Phase 1 schema — never modify them}

## Screens

| Screen id | Canonical URL | Purpose |
|---|---|---|
| {entity}-list   | /{entities}/            | List per the canonical table pattern |
| {entity}-add    | /{entities}/new         | Full-container form (empty) |
| {entity}-edit   | /{entities}/{id}/edit   | Same form, pre-filled |
| {entity}-view   | /{entities}/{id}        | Detail view (only if the plan calls for one) |

## List screen
- Columns, in order: {column → source field, format (dates: medium; status: dot+badge per the locked vocabulary)}
- Search matches: {fields}; sort allowlist: {fields}; page size: {n}
- Row link target: {entity}-view or {entity}-edit (pick per plan — state it here)

## Form
- Fields, in order: {field | input type | required? | validation rule | id ({entity}-form-field-{name})}
- Selects and their option sources: {…}
- Tabs (only if the exemplar's form has tabs): {tab → fields}

## Files (exactly these — no additions)
- public/{entities}/index.php · form.php · save.php · delete.php
- app/features/{entities}/queries.php
- app/views/{entities}/page.php · partials/table.php · row.php · form.php · saved.php

## Query functions (signatures fixed)
- find_{entities}(PDO, search='', sort='{default}', page=1): array
- find_{entity}(PDO, int id): ?array
- insert_{entity}(PDO, …explicit fields): array
- update_{entity}(PDO, int id, …explicit fields): array
- delete_{entity}(PDO, int id): bool

## Action manifest entries
- Screens: the table above, with one "when the user wants…" line each
- Actions: {action name | endpoint | params schema | undo definition | confirm? (destructive only)}

## Activity log events
- {event name per handler: {entity}_created, {entity}_updated, {entity}_deleted, screen entries}

## Status vocabulary mapping (if the entity has states)
- {state → locked color: success/warning/danger/info/secondary/dark}

## Out of scope for this slice
- {explicitly excluded things a helpful model might be tempted to add}

## Open Questions (must be EMPTY before a worker starts; workers append here when escalating)
- (none)
```
