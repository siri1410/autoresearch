# Using autoresearch with MyBotBox

autoresearch is two things: a small GPU pretraining setup (`prepare.py`,
`train.py`), and a **loop** (`program.md`): an agent edits one file, runs a
fixed-budget experiment, reads one metric, keeps or discards, repeats.

[MyBotBox](https://github.com/YarlisAISolutions/mybotbox-platform) is a
TypeScript/Bun workflow platform that calls hosted LLMs. It does not pretrain
models, so the useful part to bring over is the loop. There are three ways to
connect them, from most to least practical today.

## 1. Run the loop on docs retrieval (ready now)

MyBotBox already has the two things the loop needs:

| autoresearch | MyBotBox |
|---|---|
| `train.py`, the one file the agent edits | `apps/sat/lib/docs-eval/retrieve.ts` |
| `uv run train.py` | `cd apps/sat && bun run docs:eval` |
| `val_bpb`, lower is better | MRR / recall@k, higher is better (45-case golden set) |
| 5-minute time budget | ~45 embedding calls per run (about 20 runs/day on the free tier) |

[`program.md`](program.md) in this folder is a ready-to-use agent program for
this. Copy it into a `mybotbox-platform` checkout and tell your agent:

```
Have a look at apps/sat/lib/docs-eval/program.md and let's kick off a new experiment.
```

Watch out for:
- **Quota.** `gemini-embedding-001` on AI Studio's free tier allows 1000
  embed requests/day, and `docs:index` uses most of that. Run the loop on a
  day you are not re-indexing, or use Vertex/paid quota.
- **Query-side changes only.** Re-chunking or switching embedding models
  needs a full re-index, which is too slow and costly for an
  experiment-per-run loop. Do those by hand and re-baseline afterwards.
- **Port wins to production.** `retrieve.ts` mirrors the production reader's
  SQL on purpose. A change that only lands in the harness measures a query
  nobody runs.

## 2. Same loop, other MyBotBox targets

The pattern works anywhere there is (a) one file to edit, (b) a
deterministic, cheap scorer, and (c) a single number. Candidates in the
platform:

- **Evaluator block / agent prompts.** The `apps/sat/executor/handlers/evaluator`
  handler scores workflow outputs. Freeze a set of inputs, let the agent edit
  one prompt template, and score with the evaluator.
- **Model routing / fallback chains** (`packages/ai/src/execute`). Score
  cost × quality on a fixed task set.
- **Guardrails** (`packages/guardrails`). Score precision/recall on a
  labelled set of allowed and blocked inputs.

Each one needs its own golden set first. Without ground truth, the agent
ends up optimizing noise.

## 3. Train a small model and serve it through MyBotBox (future)

You could take a model trained here and serve it through
`@yarlisai/serving` (its `vllm` adapter). This is not wired up, and it would
take real work:

- `train.py` saves no checkpoint and uses a custom architecture (value
  embeddings, sliding-window attention, a custom tokenizer). vLLM can't load
  it as is. You would need an exporter to a HuggingFace-compatible format,
  or to train a standard architecture instead.
- At this scale (a few minutes on one GPU) the model is a research artifact,
  not something you would put behind a customer workflow. If you need a
  custom model in MyBotBox, `@yarlisai/finetune` (Gemini/Vertex tuning) is
  the practical path.

Revisit this only if you specifically need to self-host a small
domain-specific model.
