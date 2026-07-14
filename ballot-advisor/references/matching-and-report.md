# Matching & Report — interview, scoring, abstention, template

The mechanics that turn research into a defensible recommendation. The governing principle (from SKILL.md): you apply the *voter's* weighted values, never your own.

## The interview — question bank

Goal: get a usable profile in as few, well-batched questions as possible. Use `AskUserQuestion` with multiSelect where natural. Don't ask all of this if the voter volunteers a clear profile — adapt.

**Q1 — Priority issues (multiSelect).** "Which of these matter most to you this election? Pick your top few." Offer the taxonomy below. Then ask them to **rank or weight** the chosen ones (e.g., 3 = non-negotiable top, 2 = important, 1 = minor). Salience is the whole game — without it, matching collapses into counting.

**Q2 — Direction on the top issues.** For each top-weighted issue, a one-line lean. Keep options concrete and local where possible ("On housing: build more / protect neighborhood character / unsure"). Avoid abstract left/right labels — they hide disagreement.

**Q3 — Deal-breakers (non-negotiables).** "Is there any single position or trait that rules a candidate out for you no matter what else?" These are hard gates applied in scoring, not weights. Examples a voter might give: stance on a specific right, a record of corruption, anti-democracy positions, a specific local project.

**Q4 — Decision heuristics + abstention tolerance (the pivotal one).** Batch:
- Party: straight-ticket, lean-but-flexible, or fully candidate-by-candidate?
- Incumbency: reward a record, or time-for-change bias?
- Experience: value insider experience, or prefer outsiders?
- **Abstention tolerance:** "If no candidate is a good match for a given race, would you rather (a) skip that race, or (b) pick the lesser evil?" — record this; it directly flips Phase-5 outcomes.

**Q5 — Reflect back.** Summarize the profile in 4–6 lines and let them correct before you research. Cheap insurance against researching against a misread profile.

### Issue taxonomy (starter set — localize as needed)

Economy & taxes · Housing & affordability · Public safety & policing · Criminal justice / DA priorities · Education & schools · Healthcare · Environment & climate · Immigration · Abortion & reproductive rights · Guns · Civil rights / LGBTQ+ · Government reform & anti-corruption · Transportation & infrastructure · Growth & development · Homelessness · Water/utilities/special districts.

For **primaries**, add intra-party axis questions (e.g., progressive vs. establishment, MAGA vs. traditional) since party label no longer differentiates.

## Scoring — transparent, no false precision

Per candidate, per contest:

1. **Per-issue agreement.** For each of the voter's prioritized issues, assign from sourced evidence:
   - `+1` aligned with the voter's lean
   - `0` mixed, unclear, or insufficient information
   - `−1` opposed to the voter's lean
2. **Weight by salience.** Multiply each agreement by the voter's weight for that issue.
3. **Normalize to a band**, don't publish a decimal. Compute weighted agreement ÷ maximum possible weighted agreement, then map:

   | Normalized | Band |
   |---|---|
   | ≥ +0.6 | **Strong match** |
   | +0.2 to +0.6 | **Lean** |
   | −0.2 to +0.2 | **Weak / toss-up** |
   | < −0.2 | **Mismatch** |

   Bands, not numbers — the inputs are too coarse to justify precision, and false precision misleads the voter.

4. **Deal-breaker gate (overrides the band).** If a candidate violates a stated non-negotiable, cap them at **Mismatch** and state which deal-breaker tripped, regardless of computed band. Gates beat weights.

5. **Confidence (separate axis from the band).**
   - **High** — sourced evidence (ideally record, not just rhetoric) on most of the voter's top issues.
   - **Medium** — evidence on some top issues; the rest rest on stated positions or are thin.
   - **Low** — little distinguishing information; the band is a weak signal. Say so out loud.

A Strong-match/Low-confidence result is *not* a confident recommendation — the band tells you direction, confidence tells you how much to trust it. Always report both.

## Abstention decision tree (Phase 5)

Run per contest, in order:

1. **All viable candidates trip a deal-breaker** and none is meaningfully better on the voter's priorities → **Recommend non-vote.** State the shared deal-breaker.
2. **Can't differentiate** — every candidate lands Weak/toss-up at Low confidence and you genuinely found nothing distinguishing on the voter's priorities → **Recommend non-vote.** State that information was insufficient (and that this is an honest finding, not a dodge).
3. **Voter prefers skipping weak matches** (Q4) AND best available band is Weak or Mismatch → **Recommend non-vote.**
4. **A clearly better-aligned viable candidate exists** (higher band, or equal band but higher confidence / fewer reservations) → **Recommend that candidate**, with band, confidence, and reservations.
5. **Two-plus candidates genuinely tied** on band and confidence, none trips a deal-breaker → **Present as a toss-up** with the deciding considerations and let the voter break it. Don't fabricate a tiebreaker.

Notes: lesser-evil is the voter's right — when one viable option is clearly better, recommend it even if imperfect (unless Q4 said otherwise). For **uncontested** races, note the candidate is unopposed and present their brief without forcing a pick. Never abstain silently — every non-vote ships with its reason.

## Full report template

```
# Your Ballot — [Jurisdiction], [Election + Date]

Your weighted priorities: [issue (weight), issue (weight), ...]
Your deal-breakers: [...]
Your heuristics: [party stance; incumbency; abstention tolerance]

## Summary
| Contest | Recommendation | Band | Confidence |
|---|---|---|---|
| [Office] | [Candidate / Skip / Toss-up] | [band] | [H/M/L] |
| [Measure] | [Yes / No / Skip] | — | [H/M/L] |
...

## Contests
### [Office name]  ·  Recommendation: [Vote — Name / Consider skipping / Toss-up] ([band], [confidence])
Your priorities here: [the issues that bear on this race]
- **[Candidate]** ([incumbent/challenger]) — [record]/[stated] findings; alignment per issue; endorsements/funding as signal. *Sources: ...*
- **[Candidate]** — ...
**Why:** [reasoning tied to the voter's weighted priorities; note deal-breakers; explain the confidence level]

### [Ballot measure]  ·  Recommendation: [Yes / No / Skip] ([confidence])
**A YES vote means:** [plain language]. **A NO vote means:** [status quo]. **Cost/impact:** [fiscal].
**How it maps to you:** [tie to priorities; note if priorities pull both ways] *Sources: official analysis, FollowTheMoney, ...*

---
Reminder: verify candidate/measure names against your official sample ballot. Every call here is yours to override.
```

## Anti-patterns to avoid

- **Party-label collapse:** never infer a position from party alone — research the actual stance, especially down-ballot and in primaries.
- **Endorsement = position:** funding and endorsements are influence signals, not policy stances. Keep them separate.
- **Filling empty cells:** if you don't know a stance, score it `0` and write "Insufficient information." Don't infer to avoid blanks.
- **False precision:** no "87% match." Bands + confidence + reasoning.
- **Silent lean:** if you catch yourself framing one candidate more favorably than the evidence supports, rebalance before shipping.
