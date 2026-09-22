# Flashcard SRS App — Design Summary

A personal spaced-repetition web app for drilling software-development concepts. Built to
solve a specific problem: understanding a concept (e.g. variable hoisting) but forgetting it
because it isn't drilled or hit in daily work — then having to relearn it. Immediate driver is
certification prep (HashiCorp Terraform Associate now, an AWS Associate next); longer term it
doubles as a passion project and résumé piece for a pivot toward DevOps/SRE.

The guiding stance is **start simple now, leave clean seams for expansion.** Everything below is
split accordingly: the near-term build first, the ambitious stuff under *For Future Consideration*.

---

## Guiding Principles

- **Separate content from scheduling.** Card content (the questions/answers) is the durable,
  portable "product." Review/scheduling state (due dates, boxes, ease) is disposable app
  bookkeeping. They live in separate records. This is what keeps the data portable and lets the
  algorithm evolve without touching the cards.
- **Cards are clean, self-describing data.** No app-specific formatting baked in. Stable IDs so
  edits don't reset history and re-imports upsert instead of duplicating.
- **The `type`/`deck`/`tags` distinctions are data, not structure.** New card types, categories,
  or study modes become new field values — not migrations. The schema *is* the expansion seam.

---

## Card Model (v1)

A flat card object, roughly:

- `id` — stable slug (e.g. `terraform-state-purpose`); enables dedupe/upsert.
- `q` / `a` — question and answer, as **Markdown** (fenced code blocks give syntax highlighting
  for free).
- `deck` — category path, e.g. `devops/terraform`; the folder-style path *is* the taxonomy.
- `tags` — cross-cutting labels (a card can be `terraform` *and* `interview-prep`).
- `type` — **`concept`** or **`applied`** (see below).

**Two card types, one schema.** `type` affects *authoring, rendering, and filtering only* — never
scheduling, storage, or the import path.

- **Concept** — definitions and mental model ("what is hoisting in JS?"). Strong for the encoding
  stage and for knowledge that *is* the definition.
- **Applied** — transfer and doing ("what does this log?", "spot the bug", "what breaks if I swap
  `var` for `let`?"). This is the format that actually trains real-world use.

Bias new decks toward applied cards, but keep concept cards to anchor the model. An `applied` card
in v1 is just a card whose question contains a code block — **no schema change required.**

---

## Architecture (v1)

- **Client:** a PWA served as a static site, usable from both mobile and desktop browsers,
  offline-capable via a service worker.
- **Hosting:** S3 + CloudFront, provisioned with **Terraform** (the deploy doubles as real IaC
  practice for the exam).
- **Backend:** a **Lambda** (Function URL is enough) as the single write path.
- **Database:** **DynamoDB** for cross-device sync — the single source of truth. (An earlier
  git-repo-of-cards idea was dropped; Dynamo replaces it.)
- **Auth:** minimal — a Cognito user pool (free tier) or a single stored token — just enough for
  both devices to share one account.

**DynamoDB modeling — design around access patterns, not entities.** The patterns are simple: load
all cards in a category to drill, read/update review state for a card, and pull what's due. Likely a
single table holding card content plus per-user review state, keyed so both browsers read the same
rows. Card items cap at 400 KB — text cards aren't remotely close.

**Adding cards.** There is no automatic pipe from a chat to the database. Card generation produces
JSON matching the schema; the app POSTs it to the Lambda, which validates, fills defaults, and
writes with `BatchWriteItem` (chunked at 25/call). Stable IDs make this an idempotent upsert.

---

## Scheduling (v1)

- **Start with Leitner boxes** — trivial to implement, works immediately. (SM-2 and beyond are
  future work.)
- **Log every review event from day one** (append-only): `card id`, `timestamp`, `grade`, and
  `response latency`. The current box is all Leitner needs — but a future ML-based scheduler is
  *trained* on this history, and it can't be reconstructed after the fact. Cheap now, impossible
  later.
- New cards need no review record; "no review state" is treated as "new, due now."

---

## Cost (v1)

Effectively **$0/month at single-user scale.** DynamoDB (provisioned mode) and Lambda both sit in
permanent always-free tiers; S3 + CloudFront usage is negligible.

- **Set a billing alarm on day one** — AWS does not warn you before charges hit.
- **Avoid always-on services** — RDS, EC2, and especially NAT Gateway (~$32/mo) are the usual
  "why is my free project billing me" traps. Static + serverless avoids all of them.
- A custom domain is the only likely real cost (~$12–15/yr), and it's optional.

---

## Portability & Export

The cards are portable *by design* — the clean schema is the whole point. To keep that a real
guarantee (and as a backup, since Dynamo is now the only copy):

- Add a small **export endpoint** early: the Lambda scans the table and returns plain JSON (not
  DynamoDB's type-annotated export format). ~20 lines, reusing existing unmarshalling.
- Per-target adapters (e.g. Anki via TSV or `genanki`) are short scripts written once, outside the
  data. Markdown answers render to HTML in that step for Anki.
- Enable **point-in-time recovery** on the table (cheap) or drop a periodic JSON dump somewhere.
- **Review progress does not port** — intentionally. Content stays portable *because* it's clean;
  per-app scheduling history is disposable.

---

## For Future Consideration

Deliberately out of scope for v1. The v1 seams above are chosen so each of these stays *additive*
rather than a rewrite.

- **In-app chat per card** — a chat window (collapsed until the answer is revealed, so it deepens
  understanding rather than short-circuiting recall) to ask an LLM about the concept, with
  **card generation from that chat** feeding the existing import path. Note: this is the first
  feature with real, usage-based cost — keep the API key server-side, cap usage, and use a cheap
  model tier.
- **Advanced scheduling** — SM-2, then an FSRS-style ML scheduler trained on the review-event log.
  (This is the intellectually rich part and the reason to log events from day one.)
- **Richer card types** — cloze deletion, image occlusion, multiple choice, audio, LaTeX. Added as
  new `type` values; structured fields (`code`, `language`, predict-then-reveal, run-the-code) can
  hang off `applied` cards.
- **Analytics & diagnostics** — retention curves, streaks, leech detection, exam-readiness
  estimates, and **concept-vs-applied pass rate per topic** (surfaces the exact
  understand-but-can't-apply gap this app targets).
- **Offline-first multi-device editing** — genuine conflict resolution (CRDTs / a sync protocol).
  The one expansion that isn't cheap even with good seams; budget it as its own project.
- **Multi-user** — accounts, shared/published decks, a deck library, gamification.
- **Platform surface** — native mobile apps, a browser extension to make a card from any page.
- **DevOps maturity (résumé value)** — CI/CD pipeline, multiple environments, monitoring/
  observability. What makes the project read as *DevOps/SRE-minded* rather than just fullstack.
- **Linking concept ↔ applied cards** on a topic so the app can sequence them (concept first,
  then applied).
