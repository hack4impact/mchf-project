# Simple Gift → RE NXT Mapping and Real-Environment Test Plan

Related issue: [#9](https://github.com/hack4impact/mchf-project/issues/9)
Branch: `feature/9-nxt-mapping-and-test-plan`

Lab and documentation only. No real integration is proven here.

**Sources**

- **SKY**: SKY API reference (developer.sky.blackbaud.com), Gift API "Gift (Batch)" and Fundraising API, read 2026-11-07.
- **CONN**: official Power Platform connector "Blackbaud Raiser's Edge NXT", operations Add a gift to batch, Create a gift batch, List gift batches, List funds / campaigns / appeals / packages, read 2026-11-07.
- **PRD**: PRD v0.4, section 4 (what a batch can't carry stays manual for Gift Processing).
- **LAB**: Établi, Essayer l'API and `/api/renxt/openapi.json`, read 2026-11-07.

**Fact** = written in SKY or CONN. **Assumption** = seen only in the lab, or our own choice. **🚩** = no match, open question.

## 1. Field mapping

NXT fields are in **Add a gift to batch** unless noted.

| Form field (#1) | NXT field | Source | Fact / Assumption | Still to verify in the Foundation's NXT |
|---|---|---|---|---|
| Constituent | `constituent_id` = system `id` from Search for a constituent (`search_field = lookup_id`) | CONN | Fact | Exact match on the 7-digit ID |
| Gift Type | `type` (section 2) | SKY, CONN | Fact | — |
| Payment Type | `payments.payment_method` (section 2) | SKY, CONN | Fact; assumption for 3 types | — |
| Gift Date | `date` (`yyyy-MM-dd`) | CONN | Fact | — |
| Gift Amount | `amount.value` (gross amount) | CONN | Fact | — |
| Campaign | `gift_splits[].campaign_id` (system ID, not the code) | CONN | Fact | Foundation codes = `lookup_id` in NXT |
| Appeal | `gift_splits[].appeal_id` (system ID) | CONN | Fact | Same |
| Fund | `gift_splits[].fund_id` + `amount.value`, one split per fund line (system ID) | CONN | Fact | Same |
| Package (optional) | `gift_splits[].package_id` (system ID) | CONN | Fact | Same |
| Solicitor (optional) | Not sent: no field in the connector's Add a gift to batch. Entered by Gift Processing (PRD section 4, batch coverage) | SKY, CONN, PRD | Fact | What `default_fundraiser_credits` adds |
| Receipt Amount | `receipts.amount.value` | SKY, CONN | Fact | — |
| Fees | Not sent: no fee field in NXT. Entered by Gift Processing if needed (PRD section 4) | SKY, CONN, PRD | Fact | — |
| Net Amount | Not sent: computed in the app, Gift Amount − Fees | #1 data dictionary | Fact | — |
| Benefit Amount | Not sent: no benefit field in a batch gift. Entered by Gift Processing (PRD section 4) | SKY, CONN, PRD | Fact | — |

**Codes vs IDs.** The form stores codes (`HA-CARD0110`); NXT splits need system IDs. List funds / campaigns / appeals have no filter by code, return 500 records by default (5,000 max) and hide inactive records (CONN, fact). Lab IDs (assumption, real IDs will differ): HA-CARD0110 = 2, HA-ONCO0220 = 3, AC-2026-2027 = 2, MG-I-2026 = 4.

## 2. Payment types and gift types

Add a gift to batch accepts (SKY, CONN, fact):
- `payment_method`: Cash, CreditCard, PersonalCheck, DirectDebit, Other, PayPal, Venmo
- `type`: Donation, Other, GiftInKind

BusinessCheck, StockOrProperty and NonCash exist only in the lab's "Create a gift", not in SKY or the connector.

| Payment type | NXT value | Fact / Assumption |
|---|---|---|
| Cash | `Cash` | Fact |
| Personal Cheque | `PersonalCheck` | Fact |
| Business Cheque | `PersonalCheck` (no BusinessCheck in NXT) | Assumption |
| Credit Card | `CreditCard` | Fact |
| Direct Debit | `DirectDebit` | Fact |
| EFT | `Other` + `payments.reference = "EFT"` (no match in NXT) | Assumption |
| Stock donation | `Other` + `payments.reference = "Stock"` (stock is a gift type in NXT, refused by Add a gift to batch) | Assumption |
| Other | `Other` | Fact |

| Gift type | NXT value | Fact / Assumption |
|---|---|---|
| One-time gift | `Donation` | Fact |
| Pledge | 🚩 Refused by Add a gift to batch. To decide with Yelene and test in the sandbox | Fact |
| Pledge Payment | 🚩 Refused by Add a gift to batch. To decide with Yelene and test in the sandbox | Fact |

## 3. Real-environment tests

All ❌ **not proven**.

| # | Operation | Expected | Proves |
|---|---|---|---|
| R1 | Search for a constituent by the 7-digit Constituent ID (`search_field = lookup_id`, `strict_search = true`); then an unknown ID | 1 result with `id`; unknown ID: 200 empty or 404 (note which) | Lookup |
| R2 | Create a gift batch, one per coded gift, `batch_description` = submission reference | `batch_id`; batch visible in NXT | One batch per gift |
| R3 | Add a gift to batch, one fund, then two funds ($60 + $40) | Gift in the batch with the right fields; `errors` empty | Adding the gift, field mapping |
| R4 | List gift batches with `search_text` = submission reference, before and after Gift Processing approves the batch (test database only) | Note `approved`, `has_exceptions`, whether the batch stays listed | What we can see about a batch; whether a commit can be detected |
| R5 | Run R1 to R4 with a fundraiser's connection and with the flow owner's or a service account | Note which accounts work and the NXT rights needed | Permissions; per-user or service connection |
| R6 | List funds / campaigns / appeals with `limit=5000` | Foundation codes found as `lookup_id` | Codes → IDs |
