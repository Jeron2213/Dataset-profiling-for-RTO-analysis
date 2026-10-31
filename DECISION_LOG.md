# Decision log

Format: **decision** - alternatives considered - why.

## Framing
- **Judge everything in Rs per order, using one formula.** Return costs Rs 169.4 (65+65+25 + 8% x Rs180). A delivered order earns 62% margin. So shipping beats cancelling unless risk > m/(m+169): 70% at Rs 649, 77% at the dataset's average Rs 920. - Alternative: treat "risk" as the goal (minimise RTO %). - Rejected: minimising RTO % would cancel profitable orders; this was the single most important finding.
- **Margin on lost orders counts, plus a Rs 100 "future value" for first-time customers.** - Alternative: count the full Rs 300 CAC as lost on every cancelled first order. - CAC is spent whether or not we ship; the loss is the *chance to turn the buyer into a repeat customer*. Rs 100 = 25% re-buy x ~Rs 400 margin; a guess, in `config.yaml`. I also report CAC written off separately in the memo because Marketing will ask.

## Data
- **Dropped 30 duplicate order IDs (exact duplicate rows) and 1 IN_TRANSIT order; normalised "Rto", "RTO ", "delivered".** - Alternative: keep in-transit as delivered. - Unknown outcome -> would teach the model a lie.
- **Kept the 3 Rs-0 orders, 12 missing pincodes, 17 bad phones** and treated them as signals/edge cases rather than deleting. Rs-0 orders (2 of 3 delivered) look like gifts/replacements -> ship, flag for a check. Bad phones returned 47% (n=17) -> kept as a risk feature.
- **Did not use pincode as a feature.** 1,066 pincodes for 1,987 orders; none has 10+ orders. Backtest of Ops' own rule (trained Apr-Aug, tested Sep) flagged 12% of orders that returned 25.6% vs 26.1% average -> useless. City works (n 30-200 each).
- **Did not use courier or time of day.** Courier gap mostly disappears inside metro / non-metro; it's also something *we* choose, not a property of the order. Hour of day shows no clear pattern (26-32% across buckets). - Noted courier as a follow-up experiment.
- **Language (Hinglish) not treated as risk.** 31% vs 28%: tiny next to geography.

## Model
- **Logistic regression, 28 features (17 cities + 11 flags)** - alternatives: gradient boosting, LLM scoring each order. - 2,000 rows, one holdout month; I did not test boosting; with ~2,000 rows the gain would be small and it can't give clean one-line reasons. LLM scoring is slower, costs money per order, and is not reproducible. AUC 0.74 out-of-time.
- **Cities with <25 orders grouped as "other"** to avoid learning noise from 3-order cities.
- **Time-based split (train Apr-Aug, test Sep)** not random split - because in production we predict the future.

## Thresholds (`config.yaml`) - cancel 75% / confirm 45% / nudge 20%
- First guess was 60/38/26 from intuition; it barely beat shipping everything. `python run.py tune` grid-searches on **out-of-fold scores from Apr-Aug only**. The optimiser's answer: nudge at 10% (i.e. nearly everyone), confirm only at 55%+, and the cancel threshold hardly matters (almost nothing scores that high).
- **I deliberately did not ship the optimiser's answer.** Replayed on September it is *better* in every scenario (Rs ~41k/month base vs ~33.5k; ~17.5k vs ~10.1k pessimistic) but it messages 69% of COD customers vs 47%. The model prices WhatsApp at Rs 0.9 and does **not** price customer annoyance, template quality rating, or slower dispatch - and the "22% accept prepaid" input it leans on is a guess. 75/45/20 is a launch posture; loosen it toward 55/10 once the A/B test shows acceptance and complaint rates.
- Selection hygiene: tuning used Apr-Aug only. I did compare 4 hand-picked candidate settings on September too; with 4 candidates that is mild leakage, so treat the September Rs figures as indicative.
- **Cancel threshold deliberately high** because cancelling only pays above ~70% (see above).

## The Ops vs Marketing question
- **First-time customers are never auto-cancelled** for "looking risky" (`never_auto_cancel_first_time`). They get a WhatsApp confirmation. - Alternative: cancel above the threshold regardless. - Ops' version of that policy (cancel >=35% risk) lost Rs 19.4k in the September replay and wrote off 65 first-time customers; and 47% of the cancelled orders would have been delivered.
- **Junk address / fake name -> ask the customer to fix it, don't cancel.** My first version cancelled them. Backtest showed 36% of these still deliver and shipping them beats cancelling by Rs 3.4k on September. -> changed to a "send us your full address" WhatsApp. Only if the phone is also invalid (can't reach them) -> Hold.
- **Serial returners (2+ prior returns, 76% return) -> Hold COD, offer prepaid link.** Right at the break-even; low volume (3 in September); Ops needs a win and it's low risk to Marketing (they aren't new customers).
- **Pincode COD-stop rejected** (see Data).

## WhatsApp flow
- **Languages: Hinglish (Roman script) for Hindi-belt states, English elsewhere.** Alternatives: Hindi in Devanagari; regional languages. - Roman Hinglish is how these customers actually text; regional languages are a next step (needs templates + review).
- **Quiet hours 10pm-8am; message at 8am.** The 24h clock starts when sent, not when ordered.
- **Reminders at 6h and 18h; decision at 24h.** Two reminders max - more is spam and WhatsApp quality rating drops.
- **No reply: confirm flow -> cancel (but "reply YES within 48h to revive"); nudge flow -> ship as COD.** - Alternative: ship on silence for confirm too. - A confirm-bucket order is 55% risky; silence is a signal. The nudge bucket is 28% (average) so silence should not cancel it. Late YES revives the order so a sleeping genuine customer isn't lost.
- **"No" in the nudge flow = keep COD, not cancel.** They declined a discount, not the product.
- **Random reply -> one clarification; second random -> a human** (ops queue). Never loops with the bot.
- **STOP -> opt-out + human phone call.**

## Discount: Rs 32 on a Rs 649 order (5%, floor Rs 20, cap Rs 60)
- A customer who moves from COD to prepaid is worth more even *after* the discount until the discount reaches (at Rs 649): Rs 95 (20% risk), Rs 155 (30%), Rs 244 (45%). See `out/discount_math.md`. - We offer ~1/5 of the ceiling. - Higher discounts raise acceptance but we have **no data on acceptance**; 5% leaves a wide safety margin and is round enough to be credible. Plan: A/B 3% / 5% / 8% in the first 2 weeks.
- Discount is paid only by customers who accept, so a bigger offer only costs more where it works.

## Dashboard
- **One static HTML page** generated from the same code. - Alternatives: Streamlit/Flask app. - Zero dependencies, opens anywhere, founder can bookmark / email it.
- **Shows yesterday *and* last 30 days.** Yesterday has only ~10 orders; a single day's Rs figures are noise.
- **Model-health tile** (predicted vs actual return rate) so we notice drift.

## Not done / cut for time
- Live LLM path untested (no key). Real WhatsApp API. Real payment link. Shopify webhook. Regional-language templates. Auto-retraining job.
