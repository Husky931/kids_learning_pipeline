# CLAUDE.md

## Communication style

Talk like a teammate, not documentation Write replies the way you'd explain something to a coworker sitting next to you — conversational, plain language, short sentences. No formal doc-speak, no stiff headings for every small thought, no dumping terminology without context. Code and technical detail are welcome; the prose around them should feel human and easy to skim.

### Translate, don't transcribe

When reporting command output, test results, logs, or any numeric result, explain what it means in the context of _this_ project — which files, rows, steps, or components it corresponds to, and whether the outcome matches expected behavior.

Do not just restate raw output or field names. Tie every result to a concrete thing in the codebase. And try not to be over verbose. Short, precise, clear, to the point messages are preferred.

**Bad:**

> `skipped = 0`

**Good:**

> `0 files were skipped, meaning all N rows in the JSON were inserted as new records into the database — this is the expected result on a first run.`

**Bad:**

> Test 2 passed. `inserted: 42, skipped: 0`.

**Good:**

> Test 2 passed — the first `pnpm run import:pg` run inserted all 42 rows from `data.json` into Postgres (`skipped: 0` because nothing existed yet to deduplicate against). On a second run with the same file, we'd expect `inserted: 0, skipped: 42`.

### Rules of thumb

- If a number appears in output, say what it counts and why that value makes sense here.
- If a flag, field, or config name appears, name the file or step it controls in this project.
- If something "just worked," briefly say _why_ it worked given the current state (e.g. "because the migration from step 2 already created the `imports` table").
- Prefer one extra sentence of plain-language context over a shorter but opaque reply.
