# Ask-me-anything

**Every SaaS application is secretly a knowledge base.** Strip away the screens, buttons, and workflows, and what remains is a body of remembered facts and a set of questions the application can answer about them.

- A dashboard is the answer to *"how are we doing right now?"*
- An invoice screen answers *"what does this customer owe, and for what?"*
- A search box answers *"where is the thing I half-remember?"*
- Even a *write* operation — creating a record, submitting a form — is the act of teaching the knowledge base a new fact so a future question can be answered.

**A feature is a pre-packaged answer to a question somebody asks often enough to deserve a button.**

## The AMA lens

This reframing replaces the vague question *"what features should it have?"* with three sharp ones, in order:

1. **Who asks?** — the actors: every kind of person (or system) that will ever want something from this application.
2. **What do they ask?** — the question inventory: for each actor, everything they would want to ask the system, in their own words. *"Which of my rentals is due for inspection?" "Did anyone reply to my listing?" "What did we charge this customer last year?"*
3. **What must be remembered to answer?** — each question is traced back to the facts, entities, and events required to answer it. Those requirements *are* the memory model.

Run the loop both directions:

- **Question → memory:** a question that memory can't answer exposes a gap in the model. Fix the model.
- **Memory → question:** an entity nothing asks about is scope with no purpose. Cut it, or find the question that justifies it.

When the loop stops producing changes, the design is converging.

## The test of a good design

An idea is well understood when you can say:

> "Here is everyone who will ask this system anything, here is every question they'll ask, and here is what the system remembers that answers each one."

Two failure modes this test catches early:

- **The feature nobody asked for** — it exists in the design but traces to no question from any actor.
- **The question nobody can answer** — an actor will predictably ask it, but no remembered fact supports the answer. Traditional design discovers this in production; the AMA lens discovers it at the whiteboard.

## Why this makes better software *now*

The ask-me-anything frame used to be a metaphor. With AI assistants in every product, it is literal: users increasingly expect to type an arbitrary question into your app and get an answer. An application planned as a knowledge base — clean memory model, explicit question inventory — can hand both artifacts to an AI and support questions you never built screens for. The screens handle the frequent questions; the AI handles the long tail. Both draw on the same memory.
