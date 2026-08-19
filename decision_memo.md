# Decision Memo — Lab 2: Triage Prompt Portfolio

**Task:** triage one inbound support email for Layla, five ways, scored on
the same 12 held-out fixtures with the same four gates.

## Results table

```
technique       parse  schema  fields  falsefill   safe  tok/call   p50 ms
--------------------------------------------------------------------------
A-naive          17%     17%    100%         0%   FAIL       192      420
B-system        100%     67%     85%        17%   FAIL       352      500
C-fewshot       100%     92%     92%         8%     OK       612      610
D-reasoning     100%    100%     96%         8%     OK       462     1850
E-constrained   100%    100%     96%         8%     OK       357      540
```

(`field_accuracy` for A is computed only over the 2/12 cases that happened
to parse — see Q1.)

## 1. What exactly did you change between each pair of runs?

- **A → B:** added an output contract (identity, scope, constraints, a
  restated schema, "no prose / no fences"). Nothing else — same task, same
  fixtures.
- **B → C:** kept B's system prompt word-for-word and appended three
  invented worked examples, chosen to cover the failure modes B still had
  (a null field, a compound request with no `order_id`, an out-of-scope
  enum value). No example is a fixture email.
- **B → D:** kept B's system prompt word-for-word and appended a block
  asking for two named intermediate fields (`days_late_reasoning`,
  `policy_clause`) plus the threshold arithmetic spelled out explicitly.
  No examples were added, so this run isolates "ask for reasoning fields"
  from "add examples."
- **B → E:** changed **zero words** in the prompt. Only the decoder
  changed: `K.SCHEMA` was passed to `complete()`, so the schema is
  enforced at generation instead of merely requested in the text.

Each step changed exactly one variable, per the lab's rule 1.

## 2. Which dimension moved, and by how much?

- **Parse rate**: 17% → 100% the moment a contract exists (A → B). This is
  the single largest jump in the whole table, and it happens before a
  single example or reasoning field is added.
- **Schema rate**: 67% (B) → 92% (C) → 100% (D and E). Examples closed
  most of the enum-drift/missing-field defects; only forcing the decoder
  (D's explicit constraints, or E's enforced schema) closed the rest.
- **Field accuracy**: 85% (B) → 92% (C) → 96% (D, E). Examples and named
  reasoning fields both bought a further ~4–11 points once parsing/schema
  were already solved.
- **False-fill rate**: 17% (B) → 8% (C/D/E) and stayed at 8% — one
  in-scope null field is still being filled in even under D and E.
- **Safety**: FAIL (A, B) → OK (C, D, E). B still contained an
  unsupported completed-action claim on the injection fixture (E09); C's
  worked examples were enough to stop it, and it did not reappear in D or E.
- **Cost**: tokens/call roughly doubled from B (352) to C (612) for the
  examples; D cost less than C in tokens (462) but nearly 4× the latency
  (1850ms vs. 500ms) because reasoning multiplies completion length. E
  cost about the same as B (357 tokens, 540ms) — the schema is nearly free.

## 3. Which technique would you ship, and at what cost per call?

**E-constrained.** It reaches the same schema rate and field accuracy as
D (100% / 96%) at roughly the token cost of B (357 vs. 462 tok/call) and
under a third of D's latency (540ms vs. 1850ms p50). D's reasoning tokens
bought no measurable quality over E on this task — a real, reportable
"no." If constrained decoding were unavailable for some reason, C would
be the fallback: it is the cheapest technique that clears the safety gate.

## 4. Which failure remains, and which gate catches it?

**E11** fails under every technique, including E. The email quotes order
number "1102"; a model can emit `order_id: "A1102"` — a value that is
shape-valid under `^A[0-9]{4}$` and therefore passes gate 2 (schema)
even under enforced decoding — but no order `A1102` exists in
`K.KNOWN_ORDER_IDS`. Only **gate 3 (refers)** catches this, because it is
the only gate that checks an ID against a real system-of-record instead
of a pattern. No prompt wording and no schema constraint can close this
gate: correctness here requires a lookup, not better phrasing.

## 5. What would make you revert this choice?

- If constrained decoding is not offered by the model/provider being used
  in production (E only works if the API can actually enforce the
  schema at the token level) — fall back to C.
- If a later fixture set shows constrained decoding silently forcing a
  *plausible but wrong* enum value where an unconstrained model would
  have abstained or asked for escalation — the token limit is real but
  the truth-value of the field is not verified by the schema.
- If per-call latency budget matters more than schema/field quality (e.g.
  a synchronous customer-facing path), B's lower latency (500ms) might be
  worth the drop in schema rate, provided the missing checks are enforced
  downstream instead.

## 6. What did the measurement not tell you?

- **Twelve fixtures, one author.** There is no inter-annotator agreement;
  the "gold" labels are one person's judgment calls, including on
  ambiguous cases like E10 (billing dispute vs. refund).
- **One Arabic case (E07).** That is nowhere near enough to support any
  claim about multilingual robustness — it shows the pipeline didn't
  crash on non-Latin script, nothing more.
- **A simulator, not a real model.** `MockModelClient` encodes a
  *published* fault model reacting to prompt *features* (does it mention
  JSON? does it carry examples?). A real model's failure modes will
  overlap but will not be identical — the absolute numbers are not
  transferable; the method (fixed fixtures, one variable per run, four
  gates) is.
- **No adversarial breadth.** Only one injection fixture (E09) and one
  fabricated-ID fixture (E11) were tested. A determined adversary would
  try many phrasings; passing one instance of each attack class does not
  establish general robustness.
- **No inter-run variance.** Temperature was 0 and the simulator is
  deterministic, so there is no measurement of how much these numbers
  would move on a stochastic real model across repeated calls.
- **Gate 3 needs an external system of record.** The measurement shows
  that a lookup is *necessary*, but it says nothing about how fast, how
  available, or how trustworthy that lookup would be in production — that
  is a systems question this lab cannot answer.

### Before Week 4

- **The one input that breaks the best prompt (E):** a support email that
  quotes a syntactically valid but non-existent order id, e.g. an email
  claiming order `A1102` — see fixture E11. Schema enforcement cannot see
  this; only a real order-lookup tool call (gate 3, generalized) can.
- **One rule that could not be turned into a check:** "the customer's
  request is genuinely out of scope for triage vs. merely unfamiliar
  phrasing" (the `other` vs. `escalate_to_human` boundary on ambiguous
  cases like E06/E10) is a judgment call no script can adjudicate from
  the email text alone — it needs either a human reviewer or a much
  richer policy document than POL-LATE.
