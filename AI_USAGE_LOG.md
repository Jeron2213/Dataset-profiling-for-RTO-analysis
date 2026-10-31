# AI usage log

**Tool:** Claude (Sonnet 5.5) in an agentic coding session - used for all of the following. No other AI tool. *(Edit this file to match your own process before submitting; the examples below are what actually happened in the build session.)*

## What AI was used for
| Task | How |
|---|---|
| Exploring the CSV, spotting data problems | Ran pandas queries; found duplicates, mixed-case labels, junk addresses, sparse pincodes |
| Designing the Rs framework (break-even maths) | AI proposed, I checked the algebra with a unit test (`test_break_even`) |
| Writing the decision engine, risk model, WhatsApp state machine, dashboard | AI wrote; tests + backtest + reading the demo transcripts to verify |
| Drafting memo, README, this log | AI drafted from `out/findings.md` and `out/backtest_report.md` so every number traces to code |
| LLM in the product | Claude (address/name judgement) - code written but **not run live** |

## Where the AI was wrong or weak, and how it was caught

1. **First design cancelled junk-address orders.** It seemed obvious: "." as an address can't be delivered. The backtest showed 36% of them *do* deliver, and shipping them earned Rs 3.4k more than cancelling in the September replay. **Caught by:** comparing every rule against a "just ship it" baseline in Rs. **Fixed:** these orders now get a "send us your full address" WhatsApp; only unreachable ones are held.
2. **Intuition thresholds (cancel 60 / confirm 38 / nudge 26) barely beat shipping everything** - Rs 0.7k over a month, which was suspicious. **Caught by:** the same baseline comparison. Diagnosis: with a Rs 169 return cost vs Rs 400+ margin, cancelling is break-even only above ~70% risk, so a low cancel threshold destroys value. **Fixed:** grid-searched thresholds on out-of-fold training predictions (not on the test month; `python run.py tune`), moved to 75/45/20, then stress-tested under pessimistic assumptions. The optimiser wanted to message 69% of customers; I chose a more conservative 47% and said why in the decision log.
3. **Reused the wrong message in reminders.** The prepaid-nudge flow's reminders said "reply YES / NO" (copied from the confirm flow) while the nudge asks for "1 / 2". **Caught by:** reading the printed simulator transcripts, not just trusting the tests. **Fixed:** separate nudge reminder templates and a test on the nudge message.
4. **Held Rs 0 orders and every invalid-phone order as "bad data".** Checked the data: 2 of 3 Rs-0 orders were delivered and look like gifts/replacements; invalid-phone orders returned 47% but only 17 exist. **Fixed:** Rs 0 -> ship + flag; invalid phone is a risk feature and a Hold only when we also can't deliver or can't ask.
5. **Pincode-level risk (what Ops asked for) looks reasonable on a slide.** **Caught by:** counting orders per pincode (median 1) and back-testing the rule out-of-time: 25.6% returned vs 26.1% average. Rejected; used city instead.
6. **LLM path unverified.** AI wrote the Claude API call without being able to run it. **Mitigation:** the code falls back to rules on any failure, a sanity net forces obviously-short addresses to "unusable", and the README says plainly that this path wasn't exercised.

## How numbers were kept honest
Every figure in the memo comes from `python run.py findings` and `python run.py backtest` (outputs saved in `out/`). Rs impact of customer behaviour (reply rate, prepaid acceptance) is explicitly labelled as an assumption with a sensitivity table, not presented as a result.
