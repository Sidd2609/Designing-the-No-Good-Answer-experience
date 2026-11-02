# Copy deck: exact wording for every path

**Product (fictional):** Acme Support Assistant. Placeholders in `[brackets]` are filled at runtime.

## Voice rules

1. **Say what you did and didn't find.** "I couldn't find a reliable answer" beats "I don't understand".
2. **Never blame the user.** No "you need to rephrase". Offer it as an option.
3. **Every message ends with a next step.** No dead ends.
4. **Identify as an AI and don't pretend otherwise.** Don't say "I'll check with my team" unless a handoff really happens.
5. **No empty apology.** One plain "sorry" at most, and only after a second failure.
6. **Never promise what the system can't deliver.** Reply times come from the real SLA.
7. **Short.** Aim for under 40 words per message before buttons.

---

## 1. ANSWER (confident)

> [Answer text]
>
> *Source: [Article title]*
>
> **[ That solved it ]  [ Not quite ]**

*Why:* the source builds trust and makes errors checkable. "Not quite" is a cheap signal that feeds the eval set.

## 2. ANSWER_CAVEAT (near-miss)

> Here's what I found, but I'm not fully sure it covers your situation:
>
> [Answer text]
>
> *Source: [Article title]*
>
> If this doesn't match what you're seeing, I can bring in a person.
>
> **[ This helps ]  [ Not quite, talk to a person ]**

*Why:* states the uncertainty once, in plain words, and gives an exit. No hedging inside the answer itself.

## 3. CLARIFY (one question, tap to answer)

> I want to get this right, and I can read that a few ways. Which is closest?
>
> **[ [Option A] ]  [ [Option B] ]  [ [Option C] ]  [ None of these ]**

Example, *"Can I change my plan?"*:

> **[ Upgrade or downgrade my plan ]  [ Switch between monthly and annual billing ]  [ Change who's on my plan ]  [ None of these ]**

Rules: max 3 options plus "None of these"; options are the retrieved candidates, not generic; "None of these" moves to IDK; never ask twice.

## 4. IDK ("I don't know", never a dead end)

> I couldn't find a reliable answer to that in our help content, and I don't want to guess.
>
> Here's what I can do:
>
> **[ Rephrase my question ]  [ Talk to a person ]**
>
> *Closest articles, which may not answer your question:*
> - [Article 1]
> - [Article 2]

If no agents are online, replace "Talk to a person" with **[ Email the team ]**.

*Why:* "I don't want to guess" tells the user the silence is deliberate and protective. "May not answer your question" keeps the related links honest.

## 5. HANDOFF (agents online)

Lead line depends on the reason:

| Reason | Lead line |
|---|---|
| User asked for a person | "Of course. Connecting you with a person." |
| Loop / frustration | "I haven't managed to get you a good answer, and you shouldn't have to keep rephrasing. Let me bring in a person." |
| High-risk topic | "Questions about charges, security or your data need a person who can see your account." |

Then, always:

> I'm connecting you with a support agent now, and I'll pass along what you've told me so you won't need to repeat it.
>
> *Estimated wait: [live queue estimate]*
>
> **[ Connect me now ]  [ Email instead ]**

**Shown to the agent (not the customer):**

> **Question:** [verbatim] | **Topic:** [topic] | **Why escalated:** [reason] | **Bot attempts:** [n] | **Confidence:** [C], top sources: [ids] | **Customer said:** [one-line summary]

## 6. TICKET (no agents online)

> Our team isn't online right now. We're back [next opening time].
>
> I've drafted a message from our chat so you don't need to retype it. Send it, and you'll get a reply by email, usually within [real SLA].
>
> **[ Send to support ]  [ Edit first ]**
>
> *Draft:* "[Customer's question, bot's attempts, account email]"

*Why:* the pre-filled draft is what turns a failure into a decent experience. The SLA must come from real data or it must not appear.

## 7. OUT_OF_SCOPE

> That's outside what I can help with. I only cover questions about Acme and your account.
>
> Is there something about Acme I can help with?

## 8. Anti-patterns (what the deck replaces)

| Don't write | Why it fails | Instead |
|---|---|---|
| "Sorry, I didn't understand that." | Blames comprehension, gives no path | Path 4 |
| "I'm still learning!" | Excuse, not help | Path 4 |
| A confident guess with no source | Hallucination in a friendly voice | Path 2 or 4 |
| "Please contact support." | Leaves the user to find it | Path 5 or 6 |
| Asking 3 clarifying questions | Interrogation | One question, then caveat or IDK |
| "Connecting you with an agent…" with no agents online | Broken promise | Path 6 |
| "I'll look into this and get back to you" | The bot can't | Path 6 |

## 9. Accessibility and implementation notes

- Chips must be real buttons with keyboard focus and visible labels; don't rely on colour alone to show a path.
- Announce new bot messages to screen readers (`aria-live="polite"`).
- Log the **path**, not just the answer, for every turn. It's the basis of every metric.
- Localise copy per market; do not machine-translate path 4 without review.
