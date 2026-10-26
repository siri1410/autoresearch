# autoresearch for MyBotBox — docs retrieval

This is the `program.md` pattern from the root of this repo, pointed at a
different target: the documentation retrieval that MyBotBox's copilot and
orchestrator use (`docs_embeddings` in Postgres + pgvector), scored by the
platform's own offline golden-set eval.

Copy this file into a checkout of `mybotbox-platform` (e.g. as
`apps/sat/lib/docs-eval/program.md`) and point your agent at it.

## Setup

To set up a new experiment, work with the user to:

1. **Agree on a run tag**: propose a tag based on today's date (e.g. `oct26`).
   The branch `autoresearch/<tag>` must not already exist.
2. **Create the branch**: `git checkout -b autoresearch/<tag>` from current `main`.
3. **Read the in-scope files**:
   - `apps/sat/lib/docs-eval/retrieve.ts` — the file you modify.
   - `apps/sat/lib/docs-eval/run.ts`, `metrics.ts`, `types.ts` — the runner and
     metrics. Read only.
   - `apps/sat/lib/docs-eval/seed-cases.ts` — the golden set. Read only.
   - `apps/sat/scripts/run-docs-eval.ts` — the CLI entry point. Read only.
4. **Verify the environment**: `DATABASE_URL` points at a database whose
   `docs_embeddings` is populated (`bun run docs:index` has been run), and the
   embedding provider key is set. If not, tell the human.
5. **Initialize `results.tsv`** with just the header row. Leave it untracked.
6. **Confirm and go**.

## Experimentation

Each experiment is one eval run:

```bash
cd apps/sat && bun run docs:eval > run.log 2>&1
```

**What you CAN do:**
- Modify `apps/sat/lib/docs-eval/retrieve.ts`. Fair game: `DEFAULT_CHUNK_CANDIDATES`,
  how chunks collapse to documents (`dedupeToDocuments` — best chunk, sum,
  mean of top-n, reciprocal-rank fusion…), query rewriting before embedding,
  a hybrid lexical (`tsvector` / trigram) + vector score, header-text boosting.

**What you CANNOT do:**
- Modify the golden set, the metrics, or the runner. They are the ground truth.
- Change `DOCS_EMBEDDING_MODEL` or re-chunk the index. Both force a full
  `docs:index`, which nearly exhausts the daily embedding quota on its own.
- Add dependencies.
- Write to the database. The eval is read-only and must stay that way.

**The goal: highest MRR**, read from the `ranking:` line of `run.log`. Use
recall@1 as the tie-breaker. Also watch the `separation:` line — a change that
raises MRR but destroys `cleanlySeparated` (on-topic vs off-topic scores) makes
the production similarity floor unusable, so treat that as a regression.

**Budget**: every case is one embedding request. With 45 seed cases and the
AI Studio free tier capped at 1000 embed requests/day, you get roughly 20 runs
a day. Think before each run. Do not loop blindly.

**Simplicity criterion**: same as the root `program.md`. A tiny MRR gain that
adds a lot of code is not worth it; removing code at equal MRR is a win.

**The first run** is always the unmodified baseline.

## Output and logging

```bash
grep "ranking:\|separation:" run.log
```

Log each run to `results.tsv` (tab-separated):

```
commit	mrr	recall_at_1	cleanly_separated	status	description
```

1. git commit hash (short, 7 chars)
2. MRR (e.g. 0.812) — use 0.000 for crashes
3. recall@1 as a fraction (e.g. 0.733)
4. `true` / `false` from the separation line
5. status: `keep`, `discard`, or `crash`
6. short description of the experiment

## The loop

LOOP (until the human stops you or the daily quota is close):

1. Look at the git state: current branch and commit.
2. Edit `retrieve.ts` with one experimental idea.
3. `bun test apps/sat/lib/docs-eval` — the unit tests must still pass.
4. git commit.
5. Run the eval: `cd apps/sat && bun run docs:eval > run.log 2>&1`.
6. Read the results. If the `ranking:` line is missing, `tail -n 50 run.log`
   and fix or give up on the idea.
7. Record the result in `results.tsv`.
8. If MRR improved without breaking separation, keep the commit. Otherwise
   `git reset --hard` back to where you started.

## Shipping a win

`retrieve.ts` deliberately mirrors the SQL shape of the production reader
(`searchDocumentationServerTool`). A gain found here only reaches users once
the same change is applied there. When the run is done, summarize the kept
commits for the human and propose the matching change to the production
reader as a normal PR — never let the two drift apart.
