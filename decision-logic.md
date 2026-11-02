# Decision logic: what the assistant does when retrieval confidence is low

**Status:** Design spec v0.1 | **Owner:** Siddharth Agarwal
**Product (fictional):** "Acme", a subscription software company, with an AI support assistant that answers from a help-centre knowledge base (RAG).
**All thresholds and cost weights are illustrative priors to be calibrated on a labelled eval set. They are not measured.**

---

## 1. Principle

> **A wrong confident answer costs more than an honest "I don't know". But an unnecessary "I don't know" also costs something. The design job is to price both and route accordingly.**

Same shape as the human-in-the-loop project (02): **hard rules override scores**, and the user is always given a next step. The assistant never dead-ends.

## 2. The seven outcomes

| Path | When | What the user gets |
|---|---|---|
| **ANSWER** | Confident, clear winner, risk allows | The answer plus its source |
| **ANSWER_CAVEAT** | Near-miss confidence, clear winner, not high risk | The answer, flagged as possibly incomplete, plus an easy way out |
| **CLARIFY** | Two near-tied interpretations, or weak match with ambiguous intent | One question with tap-to-answer choices |
| **IDK** | Low confidence, nothing to clarify | Plain "I couldn't find a reliable answer", plus 3 next steps |
| **HANDOFF** | A person is needed and agents are online | Live agent, with the context passed along |
| **TICKET** | A person is needed and no agents are online | Pre-filled email/ticket, with the real reply expectation |
| **OUT_OF_SCOPE** | Not about the product | Polite boundary, redirect to what it can do |

## 3. Signals

| Signal | Meaning | Source |
|---|---|---|
| `R` retrieval score | Relevance of the top passage (0-1) | Retriever / reranker, calibrated |
| `G` groundedness | Share of the draft answer's claims supported by retrieved passages (0-1) | Verifier / LLM-as-judge |
| `C` confidence | `min(R, G)`: conservative on purpose | Derived |
| `margin` | Gap between top-1 and top-2 candidate (0-1) | Retriever |
| `risk` | low / medium / high topic class | Topic classifier + keyword gates |
| `acct` | Needs account data the assistant cannot see | Intent classifier |
| `amb` | Intent classifier sees several plausible intents | Intent classifier |
| `oos` | Not about the product | Scope classifier |
| `human` | User explicitly asked for a person | Phrase match |
| `frus` | Frustration cues (repeat, caps, "this is useless") | Sentiment classifier |
| `failed` | Turns in this session that ended without an answer | Session state |
| `clar` | Clarifying questions already asked | Session state |
| `online` | Agents available now | Workforce system |

**Why `min(R, G)`:** retrieval can be relevant while the generated answer drifts beyond it, and the reverse. Taking the weaker of the two means one good signal can't mask one bad one.

## 4. Where the thresholds come from: the cost-of-error ratio

Answer only when expected value is positive. If a correct answer is worth 1 and a wrong confident answer costs `c`:

```
answer if  p*1 > (1-p)*c   =>   p > c / (1 + c)
```

| Risk class | Example topics | Cost of wrong answer, `c` (relative to value of a right one) | Answer threshold |
|---|---|---|---|
| Low | Reset password, find a setting | 1.5 | **0.60** |
| Medium | Plan features, limits, integrations | 3 | **0.75** |
| High | Charges, security, legal, data deletion | 9 | **0.90** |

The `c` values are judgement calls and the point is that **they are explicit and debatable**, rather than one global threshold nobody can defend. The interactive mock lets you change them and watch the thresholds move.

A **near-miss band** sits 0.15 below each threshold: not enough to answer plainly, enough to answer with a caveat (not allowed for high risk).

## 5. Decision rules (first match wins)

**Layer 1: hard rules (confidence is ignored)**

| # | Rule | Outcome |
|---|---|---|
| 1 | Out of scope | OUT_OF_SCOPE |
| 2 | User asked for a person | Escalate |
| 3 | Frustration cue, or 2+ failed turns (loop breaker) | Escalate |
| 4 | High-risk and needs account data | Escalate |
| 5 | High-risk and `C` below its threshold | Escalate |

*Escalate* = HANDOFF if agents are online, else TICKET.

**Layer 2: confidence bands**

| # | Rule | Outcome |
|---|---|---|
| 6 | `C >= t` and `margin >= 0.10` | ANSWER |
| 7 | `C >= t`, `margin < 0.10`, no clarification asked yet | CLARIFY |
| 8 | `C >= t`, `margin < 0.10`, already clarified once | ANSWER_CAVEAT |
| 9 | `C >= t - 0.15` and `margin >= 0.10` | ANSWER_CAVEAT |
| 10 | `C >= t - 0.15`, `margin < 0.10`, no clarification yet | CLARIFY |
| 11 | `R >= 0.35`, `amb`, no clarification yet | CLARIFY |
| 12 | Otherwise | IDK |

```python
def decide(s, T):                      # T = thresholds by risk class
    C, t = min(s.R, s.G), T[s.risk]
    if s.oos:                                        return "OUT_OF_SCOPE"
    if s.human or s.frus or s.failed >= 2:           return escalate(s)
    if s.risk == "high" and (s.acct or C < t):       return escalate(s)
    if C >= t:
        if s.margin >= 0.10:                         return "ANSWER"
        return "CLARIFY" if s.clar < 1 else "ANSWER_CAVEAT"
    if C >= t - 0.15:
        if s.margin >= 0.10:                         return "ANSWER_CAVEAT"
        if s.clar < 1:                               return "CLARIFY"
    if s.R >= 0.35 and s.amb and s.clar < 1:         return "CLARIFY"
    return "IDK"

def escalate(s): return "HANDOFF" if s.online else "TICKET"
```

## 6. Design choices worth defending

1. **One clarifying question, maximum.** A second question makes the bot feel like an interrogation. After one, it answers with a caveat or says it doesn't know.
2. **Two failed turns trigger escalation.** The loop breaker. Rephrasing three times is the worst experience in support bots.
3. **High risk ignores a good score.** A 95%-confident answer about a duplicate charge still goes to a person, because the bot can't see the account.
4. **IDK is never a dead end.** Every IDK offers rephrase, closest articles, and a human route.
5. **Context travels with the handoff.** The agent sees the question, what the bot tried, and why it escalated, so the customer never repeats themselves.
6. **Be honest about the channel.** Offline hours show the real reply expectation, not "we'll get back to you soon".

## 7. Metrics and eval plan

Build a labelled set of queries, each tagged with the **expected path** (answerable / ambiguous / unanswerable / must-human / out of scope). Run the logic and read a confusion matrix of expected vs actual path.

| Metric | Why it matters | Direction |
|---|---|---|
| **Confident-wrong rate**: share of ANSWER outputs that are incorrect | The costly failure | Minimise first |
| **Unnecessary-IDK rate**: IDK where the KB did contain the answer | The opposite failure | Minimise second |
| **IDK precision**: IDK where the KB truly lacked the answer | Is "I don't know" honest? | Maximise |
| **Escalation appropriateness**: handed-off cases a person actually needed | Protects agent time | Maximise |
| **Clarify success**: clarified sessions that end in a resolved answer | Is the question earning its place? | Maximise |
| Post-IDK outcome: rephrase / human / abandon | Is the dead end avoided? | Abandon rate down |
| 7-day repeat contact on same topic | Hidden failure of "resolved" | Down |
| CSAT split by path | Does honesty hurt or help satisfaction? | Compare, don't assume |

**Do not use raw deflection rate as a headline.** It rewards the bot for answering when it shouldn't.

Calibration step: plot `C` against actual correctness on the labelled set and check that "0.75" really means roughly 75% correct. If it doesn't, fix calibration before touching thresholds.

## 8. Known gaps

- Thresholds, `c` values and the 0.10 margin are priors.
- Assumes `R` and `G` are calibrated, and often they aren't.
- No tool access: with account lookup, many "account-specific" escalations could become ANSWERs.
- Not covered: multilingual queries, voice, repeat customers, and abuse.
