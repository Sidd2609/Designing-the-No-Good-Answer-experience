# Failure is a Feature: Designing the "No Good Answer" experience

> **One-line outcome:** A 12-rule decision policy and copy deck that routes every low-confidence moment to one of seven paths (answer, caveat, clarify, "I don't know", live handoff, email, out of scope), with thresholds derived from the cost of being wrong rather than guessed.

**Role:** Product Owner (solo) | **Artifacts:** [Interactive decision mock](no_good_answer_demo.html) · [Decision logic spec](decision-logic.md) · [Copy deck](copy-deck.md) · [Medium article](medium-article.md)
**Data:** Fictional product ("Acme"), invented scenarios. No employer data.

## 1. Problem and user
- **Who is stuck:** a customer asking a support bot something its knowledge base can't answer; and the support lead who owns the cost when it bluffs.
- **Pain today:** every retrieval system eventually finds nothing relevant. Most bots then either hallucinate confidently or say "Sorry, I didn't understand" and loop.
- **Why it matters:** this moment is where trust is won or lost, and it says more about product maturity than the happy path.

## 2. The decision this informs
> "When retrieval confidence is low, should the assistant answer, ask, admit it doesn't know, or hand off, and who decides the threshold?"

## 3. Approach and trade-offs
| Choice | Options considered | Picked | Why |
|---|---|---|---|
| Threshold basis | One global cut-off / per-topic / cost-of-error | Per-risk threshold from `p > c/(1+c)` | Makes the trade-off explicit and debatable |
| Confidence signal | Retrieval score only / groundedness only / both | `min(retrieval, groundedness)` | One good signal can't hide a bad one |
| Hard rules vs scores | Scores only / rules over scores | Rules first (same principle as project 02) | A confident answer about a duplicate charge is still the wrong move |
| Clarifying | Ask freely / never / capped | One question maximum | A second question feels like an interrogation |
| Loop handling | None / escalate after N | Escalate after 2 failed turns | Rephrasing three times is the worst experience |
| Dead ends | Allow / forbid | Every path ends with a next step | Failure should still progress the task |
| Headline metric | Deflection rate / confident-wrong rate | Confident-wrong rate first | Deflection rewards bluffing |

**Left out on purpose:** the retriever itself, multilingual and voice, abuse handling, tool-based account lookup.

## 4. Results (design outputs, not measured performance)
No users or production traffic were involved. The evidence here is the policy's behaviour on 12 designed scenarios, which you can reproduce in the mock.

| Customer says | Expected path | Rule that decides |
|---|---|---|
| "How do I reset my password?" | ANSWER | Confident, clear winner |
| "Can I change my plan?" | CLARIFY | Confident but near-tied interpretations |
| "Does Pro include SSO in the EU?" | ANSWER_CAVEAT | Near-miss, clear leader |
| "Why was I charged twice?" | HANDOFF | High risk and needs account data |
| "Why was I charged twice?" (agents offline) | TICKET | Same, no agents online |
| "Do you integrate with Zorbly?" | IDK | Low confidence, nothing to clarify |
| "Delete my account and all my data" | HANDOFF | High risk and needs account data |
| "What's the weather in Leeds?" | OUT_OF_SCOPE | Not about the product |
| "I want to talk to a person" | HANDOFF | Hard rule |
| "No, that's not what I asked!" (3rd attempt) | HANDOFF | Loop breaker |
| "It's not working" | CLARIFY | Weak match, ambiguous intent |
| "It still doesn't work" (after clarifying) | IDK | Already clarified, so don't ask again |

**Derived thresholds (illustrative):** cost-of-wrong of 1.5 / 3 / 9 for low / medium / high risk gives answer thresholds of **0.60 / 0.75 / 0.90**.

**Proposed success metrics (targets to set after a shadow period):** confident-wrong rate, unnecessary-IDK rate, IDK precision, clarify success, 7-day repeat contact, CSAT by path. See [decision-logic.md](decision-logic.md#7-metrics-and-eval-plan).

## 5. Limits and honesty
- **Nothing here is measured.** Thresholds, cost weights and signal values are priors. There is no labelled eval set yet.
- **The mock has not been run in a browser.** I hand-traced all 12 scenarios through the rule order with the default thresholds (0.60 / 0.75 / 0.90) and the table above matches. Open the mock and confirm the bottom table shows the same paths before publishing.
- **Assumes calibrated scores.** Real retrievers are often over-confident, and fixing calibration comes before tuning thresholds.
- **Copy is untested with users.** Wording needs A/B testing and localisation.
- **Would flip the design:** if the assistant gains account-lookup tools, many "needs account data" escalations become answers.

## 6. What I'd do next
1. Build a labelled query set (answerable / ambiguous / unanswerable / must-human) and score the policy against it. This becomes project 04, the evals playbook.
2. Plot confidence vs actual correctness to check the thresholds mean what they claim.
3. Test the IDK and handoff copy with 5-8 users, and compare CSAT by path.

## 7. Skills demonstrated
`failure-state design` `AI product logic` `hallucination mitigation` `trust and safety trade-offs` `escalation design` `conversation design` `evals planning` `RAG product thinking`
