# Audit PDF for a Coded Gift

Related issues: [#11](https://github.com/hack4impact/mchf-project/issues/11), [#34](https://github.com/hack4impact/mchf-project/issues/34), [#57](https://github.com/hack4impact/mchf-project/issues/57)
Branches: `feature/11-audit-pdf-template`, `feature/34-audit-pdf-sprint3-fields`, `feature/57-audit-pdf-sprint4-sections`
Environment: Équipe

The PDF kept with each approved submission for audit. The flow of #15
fills the HTML template and converts it to PDF at approval.

## 1. Files

| File | Use |
|---|---|
| `docs/rebuild/audit-template.html` | The template, with one `{{...}}` placeholder per value |
| `docs/rebuild/audit-samples/single-fund.html` | Template filled with the test case, one fund line ($100), a tax receipt ($25 benefit, $75 receipt), notes and two soft credits ($60 + $40). The pledge payments and installments tables are empty |
| `docs/rebuild/audit-samples/multi-fund.html` | Template filled with the test case, two fund lines ($60 + $40), no tax receipt and no notes. The soft credits, pledge payments and installments tables are empty |
| `docs/rebuild/audit-samples/screenshots/pledge-installments.png` | Page 2 of the PDF made by the lab for `SUB-TEST57B` (#57): three installments, the other two tables empty |
| `docs/rebuild/audit-samples/screenshots/soft-credits.png` | Page 2 of the PDF made by the lab for `SUB-TEST57C` (#57): two soft credits, the other two tables empty |

## 2. Template

Plain HTML only: `<h1>`, `<h2>`, `<p>`, `<table>`, `<tr>`, `<td>`,
`<strong>`. The lab's conversion ignores CSS and images.

Sections, in order:

1. Header: "Gift coding record", submission reference, payment reference
   when there is one
2. Donor, with New donor
3. Gift: types, date, amounts
4. Coding: campaign, appeal, primary fund, package, solicitor
5. Tax receipt: tax receipt to issue, benefit amount, receipt amount
6. Notes
7. Fund lines, with their total
8. Soft credits (#57): constituent, constituent ID, amount
9. Pledge payments (#57): constituent, pledge ID, amount paid
10. Pledge installments (#57): number, due date, amount
11. People: who submitted and when, who approved and when
12. Footer: "Generated at approval. The NXT transfer result is not part of
    this record."

The three tables of #57 keep their heading row when the gift has no
entry: the PDF shows an empty table, without error. The NXT batch does
not carry soft credits or pledge payments (known limit of the lab), so
Gift Processing enters them by hand from these tables.

The PDF is made at approval, before NXT answers, so the transfer result
is never in it.

## 3. Placeholders

Each placeholder uses the internal name of the column in
`MCHF Submissions` (#1). In the flow of #15, each one is replaced with
dynamic content from **Get submission**.

| Placeholder | Source |
|---|---|
| `{{Title}}` | Submission reference, `SUB-00001` |
| `{{ConstituentID}}` | Constituent ID |
| `{{Constituent_x0020_System_x0020_I}}` | Constituent system ID |
| `{{Gift_x0020_Type}}` | Gift type |
| `{{Payment_x0020_Type}}` | Payment type |
| `{{Gift_x0020_Date}}` | Gift date |
| `{{Gift_x0020_Amount}}` | Gift amount |
| `{{Fees}}` | Fees |
| `{{Net_x0020_Amount}}` | Net amount |
| `{{Campaign}}` | Campaign |
| `{{Appeal}}` | Appeal |
| `{{Fund}}` | Primary fund. The split between funds is in the fund lines |
| `{{Package}}` | Package |
| `{{Assigned_x0020_Solicitor}}` | Assigned solicitor |
| `{{BenefitAmount}}` | Benefit amount (#21). Empty stays empty, and the row stays in the table |
| `{{ReceiptAmount}}` | Receipt amount (#21). Empty stays empty, and the row stays in the table |
| `{{ApprovedByEmail}}` | Who approved (#4) |
| `{{ApprovedAt}}` | When it was approved (#4) |

Eight placeholders are not columns. The flow builds them:

| Placeholder | Built from |
|---|---|
| `{{PaymentReferenceLine}}` | `<p><strong>Payment reference:</strong> PAY-00001</p>` when `Payment_x0020_Reference` has a value, empty otherwise |
| `{{FundLineRows}}` | One `<tr>` per row of `MCHF Fund Lines` for this submission |
| `{{FundLinesTotal}}` | Sum of the fund line amounts |
| `{{SoftCreditRows}}` | One `<tr>` per row of `MCHF - Soft Credits` for this submission (#57) |
| `{{PledgePaymentRows}}` | One `<tr>` per row of `MCHF - Pledge Payments` for this submission (#57) |
| `{{InstallmentRows}}` | One `<tr>` per row of `MCHF - Pledge Installments` for this submission, by installment number (#57) |
| `{{SubmittedBy}}` | System column Author (Créé par) of the submission |
| `{{SubmittedAt}}` | System column Created (Créé) of the submission |

For `{{PaymentReferenceLine}}`, the flow uses:

```
@{if(empty(body('Get_submission')?['Payment_x0020_Reference']), '', concat('<p><strong>Payment reference:</strong> ', body('Get_submission')?['Payment_x0020_Reference'], '</p>'))}
```

The test case does not come from the registry, so its PDF has no payment
reference line.

Four placeholders use a column, but the flow changes the value before it
goes in the HTML:

| Placeholder | Column | Value in the PDF |
|---|---|---|
| `{{NewDonor}}` | `NewDonor` (#21) | `Yes` or `No`. An empty value gives `No` |
| `{{TaxReceipt}}` | `TaxReceipt` (#21) | `Yes` or `No`. An empty value gives `No` |
| `{{Notes}}` | `Notes` (#21) | The text, with `&`, `<` and `>` escaped |
| `{{ConstituentName}}` | `ConstituentName` | The name, with `&`, `<` and `>` escaped. It is typed by hand for a new donor (#25) |

For `{{NewDonor}}` (same expression with `TaxReceipt` for `{{TaxReceipt}}`):

```
@{if(equals(body('Get_submission')?['NewDonor'], true), 'Yes', 'No')}
```

For `{{Notes}}` (same expression with `ConstituentName` for
`{{ConstituentName}}`):

```
@{replace(replace(replace(coalesce(body('Get_submission')?['Notes'], ''), '&', '&amp;'), '<', '&lt;'), '>', '&gt;')}
```

Empty fields (Package and Assigned solicitor in the test case, Benefit
amount and Receipt amount in `multi-fund.html`) stay in the table with an
empty value. Empty notes leave the Notes section with an empty paragraph.

## 4. Repeated sections

Fund lines repeat. In the flow, an **Apply to each** on the fund lines
adds one table row to a string variable with **Append to string
variable**:

```
<tr><td>@{items('Each_line')?['Title']}</td><td>@{items('Each_line')?['Amount']}</td></tr>
```

The template inserts this variable at `{{FundLineRows}}`. The same loop
adds each amount to the total.

### Sprint 4 sections (#57)

Soft credits, pledge payments and pledge installments follow the same
pattern as the fund lines, with the lists of #49. The flow reads them with
three **Get items**, placed after **Get fund lines**, each filtered on the
submission reference:

```
Submission_x0020_Reference eq '@{outputs('Get_submission')?['body/Title']}'
```

| Get items | List | Variable (string) | Loop | Row added by Append to string variable |
|---|---|---|---|---|
| `Get_soft_credits` | `MCHF - Soft Credits` | `softCreditRows` | `Each_soft_credit` | `<tr><td>name</td><td>ConstituentID</td><td>Amount</td></tr>` |
| `Get_pledge_payments` | `MCHF - Pledge Payments` | `pledgePaymentRows` | `Each_pledge_payment` | `<tr><td>PledgeConstituentName</td><td>PledgeID</td><td>Amount</td></tr>` |
| `Get_pledge_installments` | `MCHF - Pledge Installments`, `$orderby` `InstallmentNumber asc` | `installmentRows` | `Each_installment` | `<tr><td>InstallmentNumber</td><td>due date</td><td>Amount</td></tr>` |

The three variables are initialised with the other two (`rows`, `total`),
at the top of the flow, never inside a loop or a condition. The three loops
come after `Each_line`, before `PDF_Needed`. The `HTML` Compose, inside
`PDF_Needed`, has no template file: the HTML of `audit-template.html` is
written in it, with `@{variables('softCreditRows')}`,
`@{variables('pledgePaymentRows')}` and `@{variables('installmentRows')}`
where the template has its placeholders.

The names are escaped like `{{ConstituentName}}`. The due date is written
`yyyy-MM-dd`, and left empty when the item has no date:

```
@{replace(replace(replace(coalesce(items('Each_soft_credit')?['ConstituentName'], ''), '&', '&amp;'), '<', '&lt;'), '>', '&gt;')}
@{if(empty(items('Each_installment')?['DueDate']), '', formatDateTime(items('Each_installment')?['DueDate'], 'yyyy-MM-dd'))}
```

With no item, the loop does not run: the variable stays empty and the
section keeps its heading row and an empty table.

## 5. Where the PDF is kept

| Item | Value |
|---|---|
| Library | `MCHF - Audit documents` in Équipe (Listes › Nouvelle bibliothèque de documents) |
| File name | Submission reference and date, e.g. `SUB-00001-20261106-152800.pdf` |
| Link | Column `AuditPDF` on the submission (#4). Text column, the lab has no hyperlink column |
| Fallback | If the lab's file actions are not ready, the HTML goes in the `AuditHTML` column, and no PDF is claimed |

File names must be unique: Create file fails when the name already
exists. The date and time in the name make it unique.

## 6. Test flow

Flow `MCHF - Audit PDF test` in Équipe, run by hand. It converts a filled
sample to PDF, without a submission.

| Step (renamed card) | Action | Setting |
|---|---|---|
| Manually trigger a flow | Trigger | No input |
| HTML | Compose | Content of `single-fund.html` or `multi-fund.html` |
| Create HTML file | OneDrive Create file | Folder `/MCHF-Audit`, name `@concat('SUB-00001-', formatDateTime(utcNow(), 'yyyyMMdd-HHmmss'), '.html')`, content `@outputs('HTML')` |
| Wait | Delay | 1 minute |
| To PDF | OneDrive Convert file | File `@outputs('Create_HTML_file')?['body/Id']`, target type PDF |
| Create PDF file | SharePoint Create file | Site: environment of the flow, folder `/MCHF - Audit documents`, name `@concat('SUB-00001-', formatDateTime(utcNow(), 'yyyyMMdd-HHmmss'), '.pdf')`, content `@body('To_PDF')` |

## 7. Tests

All tests used the fictional test case of the issue: Camille Exemple,
Constituent ID `1234567`, one-time gift, $100, Personal Cheque, gift date
2026-11-02, campaign `AC-2026-2027`, appeal `MG-I-2026`, fund
`HA-CARD0110`.

| Test | Result |
|---|---|
| Single fund: run the test flow with `single-fund.html` | Passed. The PDF opens and shows every section, one fund line and a $100.00 total |
| Multi fund: run the test flow with `multi-fund.html` | Passed. The PDF shows `HA-CARD0110` $60.00, `HA-ONCO0220` $40.00 and a $100.00 total |
| PDF shows who submitted and when, who approved and when | Passed |
| Content of Create PDF file set to `outputs('To_PDF')` | Failed: the file does not open. Fixed with `body('To_PDF')` |
| #34: updated `single-fund.html`, converted by a test flow in a personal environment (PDF saved in OneDrive) | Passed. The PDF shows Tax receipt to issue Yes, Benefit amount $25.00, Receipt amount $75.00 and the notes |
| #34: updated `multi-fund.html`, same test flow | Passed. The PDF shows Tax receipt to issue No. The Benefit amount and Receipt amount rows stay, empty, and Notes is empty |
| #34: personal copy of `MCHF - ApproveGift`, `SUB-TEST34C` ($100, tax receipt, benefit $25, receipt $75) | Passed. The PDF in the copy's `MCHF - Audit documents` shows Benefit amount 25 and Receipt amount 75 |
| #34: name `Camille <Exemple> & Fils` and notes `Gala <table> & dinner: benefit > 0` on `SUB-TEST34C` | Passed. The PDF shows both as typed |
| #34: personal copy, `SUB-TEST34D` (new donor, no Constituent ID, no receipt, no notes, fund lines $60 + $40) | Passed. The PDF shows New donor Yes and Tax receipt to issue No. The empty rows stay |

#57 tests, in a personal environment, on copies of `MCHF - ApproveGift` and of
the lists of #49. Each test used a new submission, because the PDF is only
made when `AuditPDF` is empty. The items were created directly in the lists
and the flow was started directly. `Constituent System ID` was left empty,
so nothing is written to the shared simulated NXT.

| Test | Result |
|---|---|
| `SUB-TEST57A`: one-time gift, no soft credit, pledge payment or installment | Passed. The three tables show their heading row and no data, no error |
| `SUB-TEST57B`: pledge of $300, three installments created in the order 3, 2, 1 | Passed. The installments come out as 1, 2, 3 with due dates `2026-12-01`, `2027-12-01`, `2028-12-01` and $100 each |
| `SUB-TEST57C`: two soft credits, one named `Second <Exemple> & Fils` | Passed. Both rows are in the table and the name is shown as typed |
| `SUB-TEST57D`: pledge payment of $150 on pledge `20001` | Passed. The table shows the pledge owner, the pledge ID and the amount paid |
| Approval of the four test submissions | Known limit of the lab: status Transfer Failed (no Constituent System ID, and the simulated NXT refuses Pledge and PledgePayment). The PDF is made before this step, and it exists |

The #34 runs used a personal copy of the flow and were started directly, not
with the Approve button of the app. Their Constituent System ID was left
empty so that nothing is written to the shared simulated NXT: the recording
step answers "Not sent to NXT" and the status is Transfer Failed, as
expected.

## 8. Limits

- The lab's conversion is not proof of the real one. Conversion,
  storage, licences and permissions are checked after access.
- `utcNow()` gives UTC time. The file name is 5 hours ahead of Montréal
  in November. It stays unique, which is what matters.
- The lab adds table borders by itself. No CSS is used.
- The PDF takes two pages: the last rows of People move to page 2.
- Convert file only works on a file in the OneDrive of the flow owner.
- Use `body('To_PDF')`, not `outputs('To_PDF')`: outputs holds the whole
  response, not only the file.
- An expression typed in a field must start with `@`. Without it, the
  text is used as is.
- Notes and Constituent name are escaped, because they are typed by hand:
  a `<` would break the table of the PDF. The other text fields are not
  escaped.
- HTML ignores line breaks, so a note of several lines shows as one
  paragraph.
- The Sprint 3 fields (#34) are only in the PDFs made after the change
  request of #34 is merged in Équipe. An existing PDF is not made again.
- The Sprint 4 sections (#57) follow the same rule: only the PDFs made
  after the change request of #57 is merged have them.
- A test needs a new submission each time. With `AuditPDF` filled,
  `PDF_Needed` skips the PDF, and "Record again" does not make it again.
- The three Sprint 4 tables read the lists by `Submission_x0020_Reference`
  (text), not by the `Submission` lookup. The coding form must fill it, as
  it does for the fund lines.
- Amounts are written as stored (`100`, not `$100.00`), like the fund lines.
  The two samples show `$100.00` as the target format.

## Remaining work

- #34: propose the personal copy of `MCHF - ApproveGift` with a change
  request (Demandes de modification › Nouvelle demande). After the merge,
  approve a gift with a tax receipt in Équipe, then export the flow to the
  solution file of #15 with Solutions › Exporter.
- #57: propose the personal copy of `MCHF - ApproveGift` with a change
  request, then export the flow to
  `lab/solutions/mchf-audit-pdf-sprint4.solution.json`.

- Build the real chain in the approval flow `MCHF - ApproveGift` (#15):
  Get submission, Get fund lines, the loop, then the steps of section 6,
  then save the link in `AuditPDF`.
- Check the date and amount formats once real submissions exist.
- Test a submission that has a payment reference, to see the payment
  reference line.
