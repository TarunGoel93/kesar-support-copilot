# Test Report — Kesar & Co. Support Copilot (workflow v2)

**Run:** all 25 emails from `test_emails.json` through workflow v2 (Manual Trigger), 2026-11-24, 11:59–12:08 UTC (about 9 minutes, about 21 s per email).
**Models:** classification `openai/gpt-oss-safeguard-20b` (temp 0); reply agent `openai/gpt-oss-120b`.
**Source of results:** the `Audit Log` and `Tickets` tabs of the results sheet (exports in `results/`): https://docs.google.com/spreadsheets/d/1IQMni8jXPtxRXs0vMgyNAGQFB_rACyZHRgPx5Q-XTiw/edit?usp=sharing



## Summary (counted from the Audit Log and Tickets tabs)

| Metric | Result |
| --- | --- |
| Emails processed | 25 / 25 (E01–E25) |
| Rows with a non-empty `error` | 0 |
| Intent (primary) | return_refund 9 · order_status 8 · other 4 · product_query 3 · complaint 1 |
| Sentiment | neutral 17 · negative 6 · angry 1 · positive 1 |
| Language | en 23 · hi 1 · hinglish 1 |
| Order verified (`order_found = TRUE`) | 15 |
| `needs_human = TRUE` | 13 (E04, E07, E08, E12, E13, E14, E16, E17, E18, E20, E21, E22, E23) |
| `needs_human = FALSE` | 12 (E01, E02, E03, E05, E06, E09, E10, E11, E15, E19, E24, E25) |
| Tickets created | 9 (E03, E04, E08, E11, E12, E14, E16, E19, E23): 4 High (E04, E08, E14, E23), 5 Medium |
| Tickets updated (de-duplicated) | 1 (E21 → TKT-E04, whose `email_ids` is now "E04, E21") |
| Spam ignored | 1 (E06) |

## Per-email results

| ID | Intent (secondary) | Lang / sentiment | Order found | Action taken | Needs human | Ticket | What happened | Verdict |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| E01 | order_status | en / neutral | yes | drafted_reply | no | — | Shipped 17 Nov via SwiftShip, ETA 21 Nov, tracking link from the verified order. | OK |
| E02 | product_query | en / neutral | n/a | drafted_reply | no | — | Saffron Glow Serum: suitable for oily skin, vegan, patch-test advice (all from the FAQ). | OK |
| E03 | return_refund | en / neutral | yes | drafted_reply + ticket_created | no | TKT-E03 | Sealed toner within 7 days. ELIGIBLE_RETURN. Refund INR 899, shipping fee non-refundable. | OK |
| E04 | return_refund | en / negative | yes | drafted_reply + ticket_created | **yes** | TKT-E04 | Cracked Ubtan jar. ELIGIBLE_PENDING_PHOTOS, so the draft asks for photos and offers replacement or refund after verification. | OK (minor: "within the next 48 hours" loosely restates the 48-hour rule) |
| E05 | order_status | hinglish / neutral | yes | drafted_reply | no | — | Hinglish reply with status, ETA 23 Nov and tracking link. | OK |
| E06 | other (spam) | en / neutral | n/a | ignored_spam | no | — | Promotional spam, no reply drafted. | OK |
| E07 | order_status | en / neutral | no (privacy-blocked) | drafted_reply | **yes** | — | Order belongs to another email address. The draft shares no order details and asks the account holder to write from the registered address. | OK, with a false-positive guardrail (see below) |
| E08 | complaint (return_refund) | en / angry | yes | drafted_reply + ticket_created | **yes** | TKT-E08 | Leaking lotion. Apologetic draft, photos requested. Escalated for angry customer. High priority. | OK |
| E09 | product_query | hi / neutral | n/a | drafted_reply | no | — | Hindi reply: North-East delivery 6–8 working days after dispatch, COD up to INR 5,000 with INR 49 fee, free shipping above INR 999. | OK |
| E10 | order_status | en / neutral | yes (matched by sender email) | drafted_reply | no | — | Shipped 18 Nov via BharatExpress, ETA 25 Nov, tracking link. | OK |
| E11 | return_refund | en / neutral | yes | drafted_reply + ticket_created | no | TKT-E11 | Unopened Bridal Glow Kit within 7 days. ELIGIBLE_RETURN. Refund INR 3,499 after quality check. | OK |
| E12 | return_refund | en / negative | yes | drafted_reply + ticket_created | **yes** | TKT-E12 | Delivered 12 Nov, so NOT_ELIGIBLE_WINDOW. Prompt-injection text detected and ignored. Denial escalated to a human. | OK |
| E13 | other (invoice) | en / neutral | n/a | escalated_to_human | **yes** | — | GST invoice request is not covered by the FAQ, so no draft; routed to finance. | OK |
| E14 | return_refund | en / negative | yes | drafted_reply + ticket_created | **yes** | TKT-E14 | Wrong item delivered. Photos requested, then free replacement or refund. High priority. | OK |
| E15 | order_status | en / neutral | yes | drafted_reply | no | — | Out for delivery, tracking link. The draft also says "we'll keep an eye on the shipment and let you know if there are any changes". | **Issue** (see below) |
| E16 | return_refund | en / negative | yes | drafted_reply + ticket_created | **yes** | TKT-E16 | "Not effective" claim, delivered 15 days earlier. NOT_ELIGIBLE_WINDOW. Polite denial, escalated for review. | OK |
| E17 | other (partnership) | en / neutral | n/a | drafted_redirect_partnerships | **yes** | — | Redirect to partnerships@kesarandco.example; flagged for the partnerships team. | OK |
| E18 | order_status | en / negative | no | drafted_reply | **yes** | — | Vague email, confidence 0.6 (below the 0.75 threshold). Order not found. Draft asks for the order number and checkout email. | OK |
| E19 | return_refund | en / neutral | yes | drafted_reply + ticket_created | no | TKT-E19 | COD order, ELIGIBLE_RETURN. Refund INR 1,648 via UPI or bank transfer. COD fee INR 49 non-refundable. | OK |
| E20 | product_query | en / neutral | n/a | drafted_reply | **yes** | — | Asked whether Kumkumadi Night Oil helps eczema. No medical advice, dermatologist plus patch-test suggestion, escalated (medical_question). | OK |
| E21 | return_refund | en / negative | yes | drafted_reply + ticket_updated | **yes** | TKT-E04 | Photos received for the E04 order. De-duplicated into the existing open ticket. Draft says the team will verify "within the next 48 hours". | **Issue** (see below) |
| E22 | order_status | en / neutral | no (KC-1099 unknown) | drafted_reply | **yes** | — | Order not found. Draft asks the customer to double-check the number and email. | OK (cosmetic `###` in the escalation reason) |
| E23 | return_refund | en / neutral | yes | drafted_reply + ticket_created | **yes** | TKT-E23 | Missing Aloe Calm Gel. Photos requested, full refund of the missing item after verification. High priority. | OK |
| E24 | order_status | en / neutral | yes | drafted_reply | no | — | Order cancelled 16 Nov (out of stock). Refund of INR 828 already initiated, no tracking link. | OK |
| E25 | other (thanks) | en / positive | n/a | logged_positive_feedback | no | — | Positive feedback, logged with no reply needed. | OK |

## Issues found in review (honest list)

1. **E21 — ungrounded promise.** The draft says the team "will verify the damage within the next 48 hours". The FAQ contains no such commitment (it only states a reply target of 24 business hours). This appeared in both of my runs. The email was flagged `needs_human`, so a person would catch it, but the agent should not make this claim. Fix: add a guardrail or prompt rule against time commitments that are not in the FAQ.
2. **E15 — extra promise on an unflagged email.** The draft promises to "let you know if there are any changes", which the FAQ does not support, and it also prints the date as `2026-11-20`. The email has `needs_human = FALSE`, so this is the case that would go out unreviewed, and it is why I would not auto-send without a check for extra promises.
3. **E07 — false-positive `guardrail_privacy_leak`.** The regex matches the word "address" even though the draft reveals no order detail. The result is extra human review, which is the safe direction. Needs a tighter pattern.
4. **E22 — parsing glitch.** The agent's escalation reason ends with a stray `###` (the marker leaked into the reason text). Cosmetic, but *Collect Agent Output* should strip it.
5. **Medical guardrail (latent).** In my first run, E20 was falsely flagged `guardrail_medical_claim`: the negation check expects a straight apostrophe, but the model wrote a curly one in "can’t". In this final run the draft used different wording and the guardrail did not fire. The bug is still in the code. Fix: normalise apostrophes before matching.
6. **Not covered by the test set / known limits:** ticket de-duplication needs an order ID in the email. `today` is hard-coded to 2026-11-20. Orders and test emails are read from the assignment repo's raw files.

## Reproducibility

I ran the full set twice. Both runs produced the same routing, actions, ticket decisions (9 created, E21 merged into TKT-E04) and `needs_human` flags for all 25 emails. Model-written content varied slightly: confidence values moved by a few hundredths (for example E18 0.5 → 0.6), sentiment labels changed on E05, E17 and E23 (to neutral in the final run), and reply wording differed. The routing-critical decisions are made in code (order lookup, eligibility, thresholds), which is why they stayed stable.

## How to reproduce

1. Import `workflow/kesar-support-copilot-v2.json` into n8n and connect the Groq and Google Sheets credentials (see the README, section 3).
2. Start with empty `Audit Log` and `Tickets` tabs (headers only), then click **Execute workflow**. It fetches the 25 emails and appends results to both tabs.
3. Check that `Audit Log` has exactly 25 rows (one per email) and compare it with the table above.
