# Audit PDF for a Coded Gift

Related issue: [#11](https://github.com/hack4impact/mchf-project/issues/11)
Branch: `feature/11-audit-pdf-template`
Environment: Équipe

The PDF kept with each approved submission for audit. The flow of #15
fills the HTML template and converts it to PDF at approval.

## 1. Files

| File | Use |
|---|---|
| `docs/rebuild/audit-template.html` | The template, with one `{{...}}` placeholder per value |
| `docs/rebuild/audit-samples/single-fund.html` | Template filled with the test case, one fund line ($100) |
| `docs/rebuild/audit-samples/multi-fund.html` | Template filled with the test case, two fund lines ($60 + $40) |

## 2. Template

Plain HTML only: `<h1>`, `<h2>`, `<p>`, `<table>`, `<tr>`, `<td>`,
`<strong>`. The lab's conversion ignores CSS and images.

Sections, in order:

1. Header: "Gift coding record", submission reference, payment reference
2. Donor
3. Gift: types, date, amounts
4. Coding: campaign, appeal, fund, package, solicitor
5. Fund lines, with their total
6. People: who submitted and when, who approved and when
7. Footer: "Generated at approval. The NXT transfer result is not part of
   this record."

The PDF is made at approval, before NXT answers, so the transfer result
is never in it.

## 3. Placeholders

Each placeholder uses the internal name of the column in
`MCHF Submissions` (#1). In the flow of #15, each one is replaced with
dynamic content from **Get submission**.

| Placeholder | Source |
|---|---|
| `{{Title}}` | Submission reference, `SUB-00001` |
| `{{Payment_x0020_Reference}}` | Payment reference. The flow writes `None` when it is empty |
| `{{ConstituentID}}` | Constituent ID |
| `{{ConstituentName}}` | Constituent name |
| `{{Constituent_x0020_System_x0020_I}}` | Constituent system ID |
| `{{Gift_x0020_Type}}` | Gift type |
| `{{Payment_x0020_Type}}` | Payment type |
| `{{Gift_x0020_Date}}` | Gift date |
| `{{Gift_x0020_Amount}}` | Gift amount |
| `{{Fees}}` | Fees |
| `{{Net_x0020_Amount}}` | Net amount |
| `{{Campaign}}` | Campaign |
| `{{Appeal}}` | Appeal |
| `{{Fund}}` | Fund |
| `{{Package}}` | Package |
| `{{Assigned_x0020_Solicitor}}` | Assigned solicitor |
| `{{ApprovedByEmail}}` | Who approved (#4) |
| `{{ApprovedAt}}` | When it was approved (#4) |

Four placeholders are not columns. The flow builds them:

| Placeholder | Built from |
|---|---|
| `{{FundLineRows}}` | One `<tr>` per row of `MCHF Fund Lines` for this submission |
| `{{FundLinesTotal}}` | Sum of the fund line amounts |
| `{{SubmittedBy}}` | System column Author (Créé par) of the submission |
| `{{SubmittedAt}}` | System column Created (Créé) of the submission |

Empty fields (Package, Assigned solicitor in the test case) stay in the
table with an empty value.

## 4. Repeated sections

Fund lines repeat. In the flow, an **Apply to each** on the fund lines
adds one table row to a string variable with **Append to string
variable**:

```
<tr><td>@{items('Each_line')?['Title']}</td><td>@{items('Each_line')?['Amount']}</td></tr>
```

The template inserts this variable at `{{FundLineRows}}`. The same loop
adds each amount to the total.

Soft credits are out of the base scope (#1, section 3). When they are
added, they follow the same pattern: their own table and their own
`{{SoftCreditRows}}` variable.

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

## Remaining work

- Build the real chain in the approval flow `MCHF - ApproveGift` (#15):
  Get submission, Get fund lines, the loop, then the steps of section 6,
  then save the link in `AuditPDF`.
- Check the date and amount formats once real submissions exist.
