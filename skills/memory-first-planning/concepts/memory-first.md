# Memory-first

A **memory-first application** is designed by deciding what the system remembers before deciding what it does.

Most application design starts with features ("users can create invoices") and backfills a database to support them. Memory-first design inverts that:

> Before asking "what should the app do?", ask "what must the app know?"

The central design artifact is the **memory model**: the entities, facts, events, and relationships the system retains, named in domain language ("a Booking links a Guest to a Room over a date range") — not in SQL. Translation to a schema is a build task, and a mechanical one if the model is good.

## Why memory comes first

- **Features become derivable.** If the memory model is right, most features are a query plus a screen. If it's wrong, every feature is a fight. A feature is just a view over memory — see [ask-me-anything.md](ask-me-anything.md).
- **Scope debates get concrete.** "Should we support X?" becomes "are we willing to remember what X requires?" — a question with a visible cost.
- **The AI-era payoff.** A system whose knowledge is well-modeled is a system an AI agent can answer questions about. Memory-first applications are ask-me-anything applications by construction.

## The two memories

A memory-first application keeps two distinct kinds of memory (the technical shape of both is in [../build/tech-stack.md](../build/tech-stack.md)):

- **Record memory** — the current truth: the entities and facts of the domain, held in the system-of-record database. It answers *data* questions: "which item sold the most last week?"
- **Activity memory** — the history of what every actor did: every screen entered, every record created, every action taken, captured as logs and ingested into the memory system. It answers *activity* questions: "when did Bob start planning for Oktoberfest?"

Record memory is edited in place — it holds what *is*. Activity memory is append-only — it holds what *happened*. Both are designed deliberately: the record memory as a domain model, the activity memory as a logging discipline. Neither can be bolted on later; activity memory in particular cannot be backfilled, so it exists from the first day the application runs.
