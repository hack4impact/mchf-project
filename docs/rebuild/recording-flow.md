# Recording Flow: Approved Gift → Simulated NXT Gift Batch

Related issue: [#13](https://github.com/hack4impact/mchf-project/issues/13)
Branch: `feature/13-simulated-recording-flow`
Environment: Équipe

The flow **MCHF - Record gift in NXT** records one approved submission in the
simulated NXT: one gift batch per submission, then the gift and its fund lines
in that batch. It never changes the submission. It answers with an outcome, and
the approval flow of #15 writes the status from that answer.

Lab only. Nothing here proves the real connector, OAuth or the Foundation's
permissions (see section 8).

## 1. Contract with the approval flow (#15)

**Input** (trigger *Manually trigger a flow*): `SubmissionId`, a number, the
`ID` of the item in `MCHF Submissions`.

**Outputs** (*Respond to a PowerApp or flow*), read in #15 with
`body('Run_a_Child_Flow')?['<name>']`:

| Output | Values | Use in #15 |
|---|---|---|
| `outcome` | `success`, `failure`, `uncertain` | Which transition (T8, T9, T10 in `review-audit.md`) |
| `status` | `Added to NXT Batch` or `Transfer Failed` | Write it as is in `Status` |
| `detail` | Text | Write it in the history row's `Comment` |
| `batch_id` | NXT system ID of the batch, or empty | Optional, for the history row |

#15 calls it with **Run a Child Flow** after the audit PDF step. The flow always
answers, even when a step fails, so #15 never waits for nothing.

## 2. Outcomes

| Outcome | When | `status` | `detail` (example) |
|---|---|---|---|
| success | Add a gift to batch answered with 1 gift and no errors | Added to NXT Batch | `Gift added to the NXT gift batch "SUB-00042" (batch ID 12).` |
| failure | Something is wrong before sending: nothing is sent to NXT | Transfer Failed | `Not sent to NXT: Gift type "Pledge" cannot be sent with Add a gift to batch …` |
| failure | NXT answered an error: nothing was added | Transfer Failed | `Add a gift to batch failed (HTTP 500): … Nothing was added to batch 11.` |
| uncertain | Add a gift to batch timed out (504): the gift may be in NXT | Transfer Failed | `The gift may or may not be in NXT. Check before doing anything: in NXT, open the gift batch "SUB-00042" and look for the gift before any retry.` |

A failure or an uncertain result is never shown as success. The flow never
sends the gift a second time, and never retries on its own.

Checked before anything is sent (all problems are listed in one `detail`):

- NXT already has a batch with this submission reference **and a gift in it**
  (a second recording would create a duplicate)
- `Constituent System ID` is empty
- Gift type has no NXT value (Pledge, Pledge Payment: see section 4)
- Payment type has no NXT value
- Campaign, appeal or a fund line's fund code isn't found in NXT
- The fund lines don't add up to the Gift Amount
- Tax receipt is Yes but the Receipt Amount is empty

## 3. Steps (rebuild)

Flows › Nouveau flow, name `MCHF - Record gift in NXT`, trigger **Manually
trigger a flow** with a **Number** input named `SubmissionId`. Rename every card
as in the table: the expressions use those names. Internal column names come
from `data-dictionary.md`.

| Card (renamed) | Action | Key settings |
|---|---|---|
| Initialize outcome, detail, batch_id, splits, problems, total | Initialize variable | String `failure`; String `The recording flow stopped before calling NXT. Open its run history.`; String empty; Array `[]`; Array `[]`; Float `0` |
| Get submission | Get item | `MCHF Submissions`, Id = `SubmissionId` |
| Get fund lines | Get items | `MCHF Fund Lines`, filter `Submission_x0020_Reference eq '@{body('Get_submission')?['Title']}'` |
| Payment methods, Gift types | Compose | The two tables of section 4, as JSON (`[{"form": "eft", "nxt": "Other", "reference": "EFT"}, …]`, `form` in lower case) |
| Find payment method, Find gift type | Filter array | `item()?['form']` is equal to `toLower(trim(coalesce(body('Get_submission')?['Payment_x0020_Type'], '')))` (gift type: `Gift_x0020_Type`) |
| List funds, List campaigns, List appeals | RE NXT | Limit `5000` |
| Find campaign, Find appeal | Filter array | `item()?['lookup_id']` is equal to `trim(coalesce(body('Get_submission')?['Campaign'], ''))` (appeal: `Appeal`) |
| List gift batches | RE NXT | Search text = `@{body('Get_submission')?['Title']}`, limit `5000` |
| Batches with a gift | Filter array | `and(equals(item()?['batch_description'], body('Get_submission')?['Title']), greater(int(coalesce(item()?['number_of_gifts'], 0)), 0))` |
| Fund lines | Compose | The fund lines; when there are none, one line with the primary `Fund` and the Gift Amount (expression below) |
| Each line | Apply to each on `outputs('Fund_lines')`, concurrency 1 | Inside: **Find fund** (Filter array on `List funds`, `lookup_id` = `trim(items('Each_line')?['Title'])`); **Fund found** (Condition `length(body('Find_fund'))` is not equal to 0): yes → **Append split** to `splits`, no → **Append missing fund** to `problems` (`Fund code not found in NXT: …`); then **Add to total** (Increment variable `total` by `float(items('Each_line')?['Amount'])`) |
| Checks | Compose | An array of messages, one per check of section 2, `''` when the check passes (exact expressions in the export) |
| Problems | Filter array | From `union(outputs('Checks'), variables('problems'))`, keep `not(empty(item()))` |
| Ready | Condition | `length(body('Problems'))` is equal to 0. No → **Set problems detail**: `Not sent to NXT: @{join(body('Problems'), ' ')}` |
| Record (in Ready, yes) | Scope | **Create a gift batch** (description = `Title`, expected number 1, expected total = Gift Amount) → **Set batch_id** = `body/batch_id` → **Add a gift to batch** (below) → **Added** (Condition: `length(body/gifts)` = 1 and `length(body/errors)` = 0): yes → outcome `success` + detail, no → detail with the errors |
| Handle failure (in Ready, yes) | Scope, **Configure run after**: Record *has failed* or *has timed out* | **No answer** (Condition, *or*): `outputs('Add_a_gift_to_batch')?['statusCode']` = 504, or `actions('Add_a_gift_to_batch')?['status']` = `TimedOut`. Yes → outcome `uncertain` + the uncertain detail. No → failure detail with the HTTP code and `actions('…')?['error']?['message']` |
| Status | Compose, run after Ready *succeeded, failed, skipped, timed out* | `@{if(equals(variables('outcome'), 'success'), 'Added to NXT Batch', 'Transfer Failed')}` |
| Respond to a PowerApp or flow | Response | Text outputs `outcome`, `status`, `detail`, `batch_id` (the variables and `Status`) |

Fund lines when there are none:

```
@if(empty(outputs('Get_fund_lines')?['body/value']),
  createArray(json(concat('{"Title":"', trim(coalesce(body('Get_submission')?['Fund'], '')), '","Amount":', string(float(body('Get_submission')?['Gift_x0020_Amount'])), '}'))),
  outputs('Get_fund_lines')?['body/value'])
```

Value appended to `splits` for each line:

```json
{
  "fund_id": "@{first(body('Find_fund'))?['id']}",
  "campaign_id": "@{first(body('Find_campaign'))?['id']}",
  "appeal_id": "@{first(body('Find_appeal'))?['id']}",
  "amount": { "value": "@float(items('Each_line')?['Amount'])" }
}
```

Add a gift to batch:

| Field | Value |
|---|---|
| Batch | `@variables('batch_id')` |
| Constituent ID | `trim(Constituent_x0020_System_x0020_I)`: the system ID, not the 7-digit Constituent ID |
| Amount | `float(Gift_x0020_Amount)` |
| Date | `formatDateTime(Gift_x0020_Date, 'yyyy-MM-dd')` |
| Type | `first(body('Find_gift_type'))?['nxt']` |
| Splits | `@variables('splits')` |
| Payment method | `first(body('Find_payment_method'))?['nxt']` |
| Payment reference | `first(body('Find_payment_method'))?['reference']` (`EFT`, `Stock` or empty) |
| Comment (gift reference) | The submission reference (`Title`) |
| Receipt status | `NeedsReceipt` when `TaxReceipt` is true, otherwise `DoNotReceipt` |
| Receipt amount | `ReceiptAmount` when `TaxReceipt` is true, otherwise `0` |

`TaxReceipt` and `ReceiptAmount` are the column names planned in #21. They
are not in `MCHF Submissions` yet, so the flow sends `DoNotReceipt`. If #21
creates them with these internal names, the flow doesn't need to change.

## 4. Value mapping (from `nxt-mapping.md`)

| Payment type | NXT payment method | Payment reference |
|---|---|---|
| Cash | Cash | |
| Personal Cheque | PersonalCheck | |
| Business Cheque | PersonalCheck | |
| Credit Card | CreditCard | |
| Direct Debit | DirectDebit | |
| EFT | Other | `EFT` |
| Stock Donation | Other | `Stock` |
| Other | Other | |

| Gift type | NXT type |
|---|---|
| One-time gift | Donation |
| Pledge, Pledge Payment | Not sent: Add a gift to batch refuses them. Ends in Transfer Failed with the reason (open question in `nxt-mapping.md`) |

Codes → IDs: the form keeps codes (`HA-CARD0110`), NXT splits need system IDs.
The flow reads List funds / campaigns / appeals and matches `lookup_id`.

## 5. Uncertain result: what to check before any retry

The flow gets the same timeout when NXT saved nothing (test constituent
`9999002`) and when NXT saved the gift but the answer was lost (`9999003`). It
can't tell them apart, so a person checks. This is the procedure for Gift
Processing, for the "Transfer Failed" buttons of `review-audit.md` (T11, T12).

**In the lab**

1. Read the latest history row of the submission: it says which batch to open
   (`SUB-00042`).
2. Open **RE NXT (développeur) › Lots de dons**. Find the batch whose
   description is the submission reference. There may be more than one if the
   gift was already recorded again: check them all.
3. Click **Voir les dons** on each.
4. A gift for this donor and amount is there → click **Gift is in NXT** (T11).
   Don't record again.
5. No batch, or every batch is empty → click **Gift is not in NXT, record
   again** (T12). The flow creates a new batch.

**What the flow checks on its own**

- There is no action that reads the gifts inside a batch.
- **List gift batches** with the submission reference finds the batches. In the
  lab, each one has `number_of_gifts`. If a batch with this reference already
  holds a gift, the flow refuses to send again (failure, "not sent again").
  This blocks a duplicate when T12 is clicked by mistake on a gift that is in
  NXT.
- `number_of_gifts` is only seen in the lab (test R4 in `nxt-mapping.md`). The
  human check stays the rule.

## 6. Tests

Run on 2026-11-07 in **Équipe**, against the flow as it is there. Each test is
one submission (titled `TEST13-<ID>`) with its fund lines; the runs are in the
flow's **Historique**. The test submissions, fund lines, the parent flow of
test 14 and the receipt columns of tests 11 and 12 were deleted afterwards.

| # | Submission | Expected | Result |
|---|---|---|---|
| 1 | `1234567`, $100, fund lines $60 HA-CARD0110 + $40 HA-ONCO0220 | success, two splits | Passed: batch with one gift, splits fund 2 ($60) and fund 3 ($40) |
| 2 | `9999001` (NXT refuses the gift) | failure | Passed: `Add a gift to batch failed (HTTP 500)`, batch left empty |
| 3 | `9999002` (timeout, nothing saved) | uncertain | Passed after 10 s; the batch is empty in the simulated NXT |
| 4 | `9999003` (saved, answer lost) | uncertain | Passed after 10 s; the gift is in the batch in the simulated NXT |
| 5 | Gift type Pledge | failure, nothing sent | Passed: no batch created |
| 6 | Unknown campaign, unknown fund, lines $50 for a $80 gift | failure, all three problems listed | Passed |
| 7 | EFT, no fund lines | success, one split on the primary fund, `Other` + reference `EFT` | Passed |
| 8 | Business Cheque | success, `PersonalCheck` | Passed |
| 9 | Test 1's submission run a second time | failure, "not sent again" | Passed |
| 10 | `9999001`, then donor changed to `1234567` and run again | failure, then success (an empty batch doesn't block) | Passed |
| 11 | Tax receipt Yes, Receipt Amount $80 on $100 (columns of #21 added for the test) | success, `NeedsReceipt`, $80 | Passed |
| 12 | Tax receipt Yes, no Receipt Amount | failure, nothing sent | Passed |
| 13 | `SubmissionId` that doesn't exist | failure, the flow still answers | Passed |
| 14 | A parent flow calls it with **Run a Child Flow** (`1234567`, then `9999003`) | The parent reads `status`, `outcome`, `detail` | Passed: `Added to NXT Batch`, then `Transfer Failed` / `uncertain` after the timeout |

To repeat a test: create a submission (Nouvel élément) with `Title` = its
`SUB-000NN`, a `Constituent System ID` (`11001` for `1234567`, `11002` to
`11004` for `9999001` to `9999003`), codes `AC-2026-2027` / `MG-I-2026` /
`HA-CARD0110`, and its fund lines with the same `Submission Reference`. Then
**Tester** with its `ID`.

The fallback (*Create a gift* instead of the batch) was not needed: the batch
actions are in the lab.

## 7. Where it lives

- The flow is in **Équipe**, turned on, owned by Jérémie. It calls NXT with its
  owner's RE NXT connection (**Connexions**): if the owner changes, the new
  owner needs one.
- #15 can call it now with **Run a Child Flow**.
- `lab/solutions/mchf-record-gift.solution.json` is a backup copy (Solutions ›
  Importer). Importing it into an environment that already has `MCHF
  Submissions` and `MCHF Fund Lines` reuses them and adds nothing; the flow
  comes in turned off.
- The test batches (`TEST-13` and `TEST13-3` to `TEST13-15`, batches 1001 to
  1012) stay in the simulated NXT. A submission with one of those titles
  would be refused as a duplicate; real submissions (`SUB-…`) are not
  affected.

## 8. Limits

- Lab only: the real connector, OAuth, the Foundation's permissions and the
  account the integration runs as are not proven (tests R1 to R6 in
  `nxt-mapping.md`).
- `number_of_gifts` on List gift batches is seen in the lab only (assumption
  until R4).
- Package is not sent: the lab has no List packages to turn the code into an
  ID.
- Solicitor, fees and benefit amount are not sent (`nxt-mapping.md`).
- Pledges and pledge payments end in Transfer Failed until the question in
  `nxt-mapping.md` is answered.
- When Add a gift to batch fails, the empty batch stays in NXT. A retry
  creates a new batch.
- If Create a gift batch itself times out, no gift was sent (failure), but an
  empty batch may exist.
- The flow doesn't write to the submission or the history: #15 does.

## Deliverables

- Rebuild document: `docs/rebuild/recording-flow.md`
- Lab export of the flow and the two lists it reads:
  `lab/solutions/mchf-record-gift.solution.json`
- The flow `MCHF - Record gift in NXT` is in Équipe, turned on.
- Branch: `feature/13-simulated-recording-flow`

## Remaining work

- Call the flow from the approval flow `MCHF - ApproveGift` (#15) with **Run a
  Child Flow**, after the audit PDF step, and write `status` and `detail` from
  its answer (section 1).
- Build the "Transfer Failed" buttons of the review screen (T11, T12) with the
  check of section 5.
- Decide pledges and pledge payments with Yelene (`nxt-mapping.md`).
