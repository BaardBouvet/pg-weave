# AGENTS.md

Guidelines for anyone working on this project.

## ADRs (`plans/adr/`)

- Keep ADRs **concise** — aim for ~20-30 lines. Context, Decision, Rationale (bullet points), Consequences.
- Don't repeat information already captured in other ADRs — link to them instead.
- Use the format: title, Status, Date, Context, Decision, Rationale, Consequences.
- **Modifying existing ADRs is fine** as long as implementation has not started. Once code lands, supersede with a new ADR instead.

## plans/PLAN.md

- The main plan should reference ADRs rather than duplicating their content.
- Keep decision rationale in the ADRs.

## End-user documentation

- Treat `plans/` as internal construction material.
- Do **not** reference `plans/` (or ADR history) from end-user docs such as the root `README.md`.
- Keep root product docs focused on user-facing vision, examples, and stable reference documentation.
- Document only decisions from **Accepted** ADRs in end-user docs (`README.md`, `docs/`).
- Do **not** include syntax or features that are still `Proposed`; keep those in planning docs until accepted.
