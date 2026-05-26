# Prompt Rescue — Fix Log

Tracking each prompt change against the eval. The **eval is correct and unchanged**; we only fix the prompt.

**Baseline (broken prompt): 6/21 (29%)**

Failure clusters at baseline:
- Hallucinated entities (11 cases) — invented product names, user counts, error codes
- Wrong priority (6 cases) — tone/urgency drives the call instead of business impact
- Broken JSON (1 case) — Case 6 returned no parseable object

---

## Fix #1 — Verbatim entity extraction

**Change:** Added an `ENTITY EXTRACTION RULES` block: extract entity values verbatim from the ticket, never paraphrase/rename/normalize, use `null` when no explicit value is stated.

**Why:** The agent was rewriting source phrasing (e.g. "CRM tool" → "CRM Platform"), which the eval's hallucination guard correctly rejects because the value isn't derivable from the input.

**Result:** 6/21 → **14/21 (+8, 67%)**. Zero regressions.

**Implication:** A single grounding rule cleared 9 of 11 hallucination failures (Cases 1, 4, 6, 7, 8, 10, 15, 16 + Case 1 target). Grounding entities to source text is the highest-leverage fix for this prompt. Two residual entity cases remain (5, 19) where the model summarizes counts into a sentence instead of the literal number.

---

## Fix #2 — Priority on business impact (with P3/P4 boundary)

**Change:** Added `PRIORITY RULES`: classify on business impact only (ignore tone/urgency/caps/threats); feature request = P4 regardless of phrasing; security/PII = P1 even for one user; P3 = something is broken (even if minor), P4 = feature request or purely cosmetic.

**Why:** The agent escalated on urgency words (Case 11 SSO request → P1) and, after a first pass, over-downgraded vague tickets to P4. The P3/P4 boundary clause fixed the over-downgrade.

**Result:** 14/21 → **15/21**. Fixed Cases 9, 11, 21; one regression (Case 5 under-escalates).

**Implication:** Decoupling priority from tone is correct, but the hard cases are now genuine judgment calls going *both* directions (under- and over-escalation), not tone-driven errors.

---

## Fix #3 — NEVER block

**Change:** Added a `NEVER` section: output only the JSON object (no prose/markdown), never infer a count ("several"/"all" → null), never promise to "fix" a P4, never escalate on tone.

**Why:** Consolidates prohibitions and targets failure modes no positive rule covered — JSON discipline (Case 6) and count fabrication (Case 19).

**Result:** **15–16/21** (~74%). Case 19 now passes stably; Case 6 JSON improved but remains flaky.

**Implication:** Explicit NEVER constraints reliably kill specific failure modes (count fabrication). JSON output is still non-deterministic — likely needs prefilling or a stricter output contract, not more prose rules.

---

## Fix #4 — Response rules (mirror the judge's criteria)

**Change:** Added `RESPONSE RULES`: address the specific issue(s) and acknowledge business impact; ask for details when vague; for a P4 feature request, acknowledge the need and share with product *without* promising a fix/timeline; stay professional.

**Why:** The NEVER rule made P4 responses too dismissive (Case 11). The judge wants impact acknowledged *and* no fix promised — both halves.

**Result:** **17–19/21** (81–90%). Case 11 now passes (~half the time); audited responses for Cases 5/8 stabilized.

**Implication:** Aligning the prompt's response guidance with the judge's rubric is high-leverage. Case 11's residual flakiness is judge variance on "enough" severity acknowledgement.

---

## Fix #5 — Priority boundary examples (P1/P2, P3/P4)

**Change:** Sharpened `PRIORITY RULES` with concrete examples — P1 = down/blocked/data-loss/security; P2 = degraded-but-up; any defect (incl. cosmetic/display bugs like truncated text) is at least P3; P4 only for feature requests/aesthetic preferences.

**Why:** Cases 3 (truncated tooltip filed P4) and 13 (severe slowness escalated to P1) were stable misses needing examples, not abstract rules.

**Result:** Stable **19/21** in both runs. Fixed Cases 3 and 13; stabilized Case 8. **Regressed Case 21** (P2 vs P3) — the broadened P1/P2 wording over-escalated an intermittent, workaround-able issue.

**Implication:** Examples fix borderline calls cleanly, but a too-broad rule creates a new boundary error nearby (whack-a-mole). Needs a P2-vs-P3 clarifier (intermittent + workaround + few/unspecified users = P3).

---

## Fix #6 — P2/P3 clarifier (recover Case 21)

**Change:** Added a P2-vs-P3 line: P2 = serious, consistent failure with significant impact; intermittent issues with a workaround, or affecting few/unspecified users, are P3 even if a major feature is involved.

**Why:** Surgically reverse the Fix #5 regression on Case 21 without disturbing Case 13.

**Result:** **20–21/21** (95–100%). Case 21 recovered; Case 13 stayed P2; no new regressions. Case 11 is the only residual.

**Implication:** A narrow, well-scoped clarifier fixes a regression without ripple. Boundaries are best tuned with a positive rule *plus* an exception clause.

---

## Score progression

6/21 → 14/21 (#1) → 15/21 (#2) → 15–16/21 (#3) → 17–19/21 (#4) → 19/21 (#5) → 20–21/21 (#6)

## Case 11 — investigated (was mischaracterized)

Earlier called a "~50% judge-variance ceiling." A 20-sample probe disproved that:
- Current prompt → Case 11: **10/10 PASS**
- A fixed ideal P4 response → judge: **10/10 PASS**

True pass rate is ~90%+, not a coin flip. Fix #4 (response rules) actually solved it — every response lands the judge's wanted pattern ("feature request, not a bug"; acknowledge need; no fix promise). The rare miss is a judge sampling-temperature tail draw (eval-side, off-limits), not a prompt defect.

**Lesson:** don't call something a "ceiling" from 4 samples — measure the rate first.

## Final state

Prompt reliably scores **21/21**, with an occasional **20/21** from a rare judge tail draw on Case 11. Eval unchanged throughout.
