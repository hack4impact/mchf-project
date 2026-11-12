# Review App, Approval Flow and Simulated Transfer

Related issue: [#15](https://github.com/hack4impact/mchf-project/issues/15)
Branch: `feature/15-correction-loop-and-transfer`
Environment: Équipe

Gift Processing reviews a submission in the app **MCHF - Review**: claim,
return with a comment, reject, approve or cancel. Approve calls the flow
**MCHF - ApproveGift**, which saves the audit PDF (#11), then calls the
recording flow of #13 and writes Added to NXT Batch or Transfer Failed.
A second screen of the same app lets the fundraiser fix a returned
submission and resubmit it.

The statuses and transitions come from `review-audit.md` (#4). The PDF
template comes from `audit-pdf.md` (#11). The recording flow is described
in `recording-flow.md` (#13).

Lab only. Nothing here proves the real conversion, the real NXT connector
or the Foundation's permissions.

## 1. Lists and names

| Display name in the app | Use |
|---|---|
| `'MCHF Submissions'` | The submissions. Status column: `'Processing status'` |
| `'MCHF Fund Lines'` | Read by the flow for the PDF |
| `'MCHF - Submission History'` | One row per action (#4, section 7) |
| `'MCHF - Payment Log'` | The linked payment. Its `PAY-…` reference is in the column `Reference`, not `Title` |
| `'MCHF-ApproveGift'` | The approval flow, called with `.Run` |

The app uses display names. The flow uses internal names
(`Submission_x0020_Reference`, `Payment_x0020_Reference`, see
`data-dictionary.md`).

## 2. Approval flow: MCHF - ApproveGift

Flows › Nouveau flow in Équipe, name `MCHF - ApproveGift`, trigger **When
Power Apps calls a flow (V2)** with a **Nombre** input named
`SubmissionID`. Picked with ⚡, it is written `@triggerBody()?['number']`.
In the app the flow appears as `'MCHF-ApproveGift'`.

Rename every card as in the table: the expressions use those names. A card
added in the wrong place (inside or outside the loop) cannot be moved in
the lab: delete it and add it again with the right **+**.

| Card (renamed) | Action | Key settings |
|---|---|---|
| Answer the app | Respond to a PowerApp or flow | First action. One text output, value `started`. The app gets its answer at once and doesn't wait for the Delay |
| Get submission | Get item | `MCHF Submissions`, Id = `SubmissionID` (picked with ⚡) |
| Get fund lines | Get items | `MCHF Fund Lines`, filter `Submission_x0020_Reference eq '@{outputs('Get_submission')?['body/Title']}'` (no space before the closing `'`) |
| Initialize variable | Initialize variable | `rows`, String, empty |
| Initialize variable 2 | Initialize variable | `total`, Float, `0` |
| Each line | Apply to each | `@outputs('Get_fund_lines')?['body/value']` |
| Append to string variable (in Each line) | Append to string variable | `rows`, value `<tr><td>@{items('Each_line')?['Title']}</td><td>@{items('Each_line')?['Amount']}</td></tr>` |
| Increment variable (in Each line) | Increment variable | `total`, value `@items('Each_line')?['Amount']` (no braces, so the value stays a number) |
| HTML | Compose | The template of #11 with every placeholder replaced (section 3). After the loop, not inside it |
| Create HTML file | OneDrive for Business, Create file | Folder `/MCHF-Audit`, name `@{body('Get_submission')?['Title']}-@{formatDateTime(utcNow(), 'yyyyMMdd-HHmmss')}.html`, content `@outputs('HTML')` |
| Wait | Delay | 1 minute |
| To PDF | OneDrive for Business, Convert file | File `@outputs('Create_HTML_file')?['body/Id']`, target type PDF |
| Create PDF file | SharePoint, Create file | Site: environment of the flow, folder `/MCHF - Audit documents`, name `@{outputs('Create_HTML_file')?['body/NameNoExt']}.pdf`, content `@body('To_PDF')` |
| Get link | Get file properties | Library `MCHF - Audit documents`, Id `@outputs('Create_PDF_file')?['body/ItemId']` |
| Save link | Update item | `MCHF Submissions`, Id = `SubmissionID`, Audit PDF = `@{outputs('Get_link')?['body/{Link}']}`. Every other field empty |
| Record in NXT | Run a Child Flow | Child flow `MCHF - Record gift in NXT` (#13), its input SubmissionId = `SubmissionID` of the trigger |
| Added | Condition | `@equals(body('Record_in_NXT')?['outcome'], 'success')`. **Configure run after**: Record in NXT *is successful*, *has failed*, *has timed out* |
| Set added (Added, yes) | Update item | `MCHF Submissions`, Id = `SubmissionID`, Processing status = `Added to NXT Batch` |
| History added (Added, yes) | Create item | `MCHF - Submission History`, see below |
| Set failed (Added, no) | Update item | `MCHF Submissions`, Id = `SubmissionID`, Processing status = `Transfer Failed` |
| History failed (Added, no) | Create item | `MCHF - Submission History`, see below |

The two history rows:

| Field | History added | History failed |
|---|---|---|
| Title | `Recording result` | `Recording result` |
| Submission Reference | `@outputs('Get_submission')?['body/Title']` | same |
| From Status | `Recording in Progress` | `Recording in Progress` |
| To Status | `Added to NXT Batch` | `Transfer Failed` |
| Actor Email | `System` | `System` |
| Comment | `@{body('Record_in_NXT')?['detail']}` | `@{coalesce(body('Record_in_NXT')?['detail'], 'The recording flow did not answer. Check the gift batch in NXT before any retry.')}` |
| Action At | `@{utcNow()}` | `@{utcNow()}` |

Only `success` leads to Added to NXT Batch. A failure, an uncertain result
(NXT timed out) or a recording flow that fails or times out all go to the
no branch, so a failed transfer is never shown as Added to NXT Batch (T8 to
T10 in `review-audit.md`). The uncertain case keeps the detail of #13,
which asks to check the batch in NXT before any retry.

## 3. HTML of the PDF

The template `audit-template.html` (#11) with each placeholder replaced by
an expression. Content of the **HTML** card:

```
<h1>Gift coding record</h1>
<p><strong>Submission reference:</strong> @{body('Get_submission')?['Title']}</p>
@{if(empty(body('Get_submission')?['Payment_x0020_Reference']), '', concat('<p><strong>Payment reference:</strong> ', body('Get_submission')?['Payment_x0020_Reference'], '</p>'))}

<h2>Donor</h2>
<table>
<tr><td><strong>Constituent ID</strong></td><td>@{body('Get_submission')?['ConstituentID']}</td></tr>
<tr><td><strong>Constituent name</strong></td><td>@{body('Get_submission')?['ConstituentName']}</td></tr>
<tr><td><strong>Constituent system ID</strong></td><td>@{body('Get_submission')?['Constituent_x0020_System_x0020_I']}</td></tr>
</table>

<h2>Gift</h2>
<table>
<tr><td><strong>Gift type</strong></td><td>@{body('Get_submission')?['Gift_x0020_Type']}</td></tr>
<tr><td><strong>Payment type</strong></td><td>@{body('Get_submission')?['Payment_x0020_Type']}</td></tr>
<tr><td><strong>Gift date</strong></td><td>@{body('Get_submission')?['Gift_x0020_Date']}</td></tr>
<tr><td><strong>Gift amount</strong></td><td>@{body('Get_submission')?['Gift_x0020_Amount']}</td></tr>
<tr><td><strong>Fees</strong></td><td>@{body('Get_submission')?['Fees']}</td></tr>
<tr><td><strong>Net amount</strong></td><td>@{body('Get_submission')?['Net_x0020_Amount']}</td></tr>
</table>

<h2>Coding</h2>
<table>
<tr><td><strong>Campaign</strong></td><td>@{body('Get_submission')?['Campaign']}</td></tr>
<tr><td><strong>Appeal</strong></td><td>@{body('Get_submission')?['Appeal']}</td></tr>
<tr><td><strong>Primary fund</strong></td><td>@{body('Get_submission')?['Fund']}</td></tr>
<tr><td><strong>Package</strong></td><td>@{body('Get_submission')?['Package']}</td></tr>
<tr><td><strong>Assigned solicitor</strong></td><td>@{body('Get_submission')?['Assigned_x0020_Solicitor']}</td></tr>
</table>

<h2>Fund lines</h2>
<table>
<tr><td><strong>Fund</strong></td><td><strong>Amount</strong></td></tr>
@{variables('rows')}
<tr><td><strong>Total</strong></td><td><strong>@{variables('total')}</strong></td></tr>
</table>

<h2>People</h2>
<table>
<tr><td><strong>Submitted by</strong></td><td>@{body('Get_submission')?['Author']?['Email']}</td></tr>
<tr><td><strong>Submitted at</strong></td><td>@{body('Get_submission')?['Created']}</td></tr>
<tr><td><strong>Approved by</strong></td><td>@{body('Get_submission')?['ApprovedByEmail']}</td></tr>
<tr><td><strong>Approved at</strong></td><td>@{body('Get_submission')?['ApprovedAt']}</td></tr>
</table>

<p>Generated at approval. The NXT transfer result is not part of this record.</p>
```

Approved by and Approved at are written by the app before it calls the
flow, so they are in the PDF.

## 4. Review app: MCHF - Review

Applications › Nouvelle application in Équipe: `MCHF - Review`, tablet,
formulas in English. Données: the four lists of section 1 and the flow.

The app is in the lab export (Deliverables), but it can't be exported to
the real Power Apps yet: rebuild it from this section.

### Controls

| Control | Settings |
|---|---|
| Screen `scrReview` | |
| Gallery `galSubs` | X 40, Y 40, Width 560, Height 690. Items `Sort('MCHF Submissions', ID, SortOrder.Descending)`. OnSelect `Set(varSel, galSubs.Selected)` |
| Label `Title1` (in galSubs) | Text `ThisItem.Title` |
| Label `Subtitle1` (in galSubs) | Text `ThisItem.'Constituent Name' & " · " & Text(ThisItem.'Gift Amount') & " $ · " & ThisItem.'Processing status'.Value` |
| Label `lblDetail` | X 640, Y 40, Width 680, Height 300. Text below |
| Text input `txtComment` | X 640, Y 360, Width 680, Height 100. Mode `TextMode.MultiLine`, Default `""`, HintText `"Comment for the fundraiser (required for Return and Reject)"` |
| Buttons | Y 480, Width 120, Height 40. Claim and Return at X 640 (never visible together), Reject X 780, Approve X 920, Cancel X 1060 |

In the lab, `ThisItem` isn't recognized in the OnSelect of the gallery
itself, only in the controls of its template. The gallery uses
`galSubs.Selected`.

`lblDetail.Text`:

```
If(IsBlank(varSel), "Select a submission.",
  "Reference: " & varSel.Title & Char(10) &
  "Donor: " & varSel.'Constituent Name' & " (" & varSel.'Constituent ID' & ")" & Char(10) &
  "Gift: " & Text(varSel.'Gift Amount') & " $, " & varSel.'Payment Type' & Char(10) &
  "Coding: " & varSel.Campaign & " / " & varSel.Appeal & " / " & varSel.Fund & Char(10) &
  "Payment reference: " & varSel.'Payment Reference' & Char(10) &
  "Status: " & varSel.'Processing status'.Value & Char(10) &
  "Review comment: " & varSel.'Review Comment')
```

### Buttons

Each button is visible only in its From statuses (`review-audit.md`,
section 4). Each one writes a history row, updates the submission, and
shows a notification.

| Button | Visible | Transition |
|---|---|---|
| `btnClaim` | `varSel.'Processing status'.Value = "Submitted"` | T2 |
| `btnReturn` | `varSel.'Processing status'.Value = "Under Review"` | T4 |
| `btnReject` | `varSel.'Processing status'.Value = "Under Review"` | T6 |
| `btnApprove` | `varSel.'Processing status'.Value = "Under Review"` | T7 |
| `btnCancel` | `varSel.'Processing status'.Value in ["Submitted", "Under Review", "Needs Information"]` | T13 |

`btnClaim.OnSelect`. The first check keeps a second reviewer from taking a
submission someone else already claimed:

```
If(LookUp('MCHF Submissions', ID = varSel.ID).'Processing status'.Value <> "Submitted",
  Notify("This submission was already claimed. Reopen the app.", NotificationType.Error),
  Patch('MCHF - Submission History', Defaults('MCHF - Submission History'), {Title: "Claim",
    'Submission Reference': varSel.Title, 'From Status': "Submitted", 'To Status': "Under Review",
    'Actor Email': User().Email, 'Action At': Now()});
  Set(varSel, Patch('MCHF Submissions', varSel, {'Processing status': {Value: "Under Review"},
    'Reviewer Email': User().Email}));
  Notify("Claimed.", NotificationType.Success))
```

`btnReturn.OnSelect`:

```
If(IsBlank(txtComment.Text),
  Notify("Write a comment for the fundraiser.", NotificationType.Error),
  Patch('MCHF - Submission History', Defaults('MCHF - Submission History'), {Title: "Return",
    'Submission Reference': varSel.Title, 'From Status': "Under Review", 'To Status': "Needs Information",
    Comment: txtComment.Text, 'Actor Email': User().Email, 'Action At': Now()});
  Set(varSel, Patch('MCHF Submissions', varSel, {'Processing status': {Value: "Needs Information"},
    'Review Comment': txtComment.Text}));
  Reset(txtComment);
  Notify("Returned to the fundraiser.", NotificationType.Success))
```

`btnReject.OnSelect`. The linked payment goes back to Unclaimed (#3):

```
If(IsBlank(txtComment.Text),
  Notify("Write the reason for the rejection.", NotificationType.Error),
  Patch('MCHF - Submission History', Defaults('MCHF - Submission History'), {Title: "Reject",
    'Submission Reference': varSel.Title, 'From Status': "Under Review", 'To Status': "Rejected",
    Comment: txtComment.Text, 'Actor Email': User().Email, 'Action At': Now()});
  Set(varSel, Patch('MCHF Submissions', varSel, {'Processing status': {Value: "Rejected"},
    'Review Comment': txtComment.Text}));
  If(!IsBlank(varSel.'Payment Reference'),
    Patch('MCHF - Payment Log', LookUp('MCHF - Payment Log', Reference = varSel.'Payment Reference'),
      {Status: {Value: "Unclaimed"}, 'Claimed By Name': "", 'Submission Reference': ""}));
  Reset(txtComment);
  Notify("Rejected.", NotificationType.Success))
```

`btnApprove.OnSelect`. Approved by and Approved at are saved before the
flow runs, so the PDF shows them. The linked payment becomes Coded:

```
Patch('MCHF - Submission History', Defaults('MCHF - Submission History'), {Title: "Approve",
  'Submission Reference': varSel.Title, 'From Status': "Under Review", 'To Status': "Recording in Progress",
  'Actor Email': User().Email, 'Action At': Now()});
Set(varSel, Patch('MCHF Submissions', varSel, {'Processing status': {Value: "Recording in Progress"},
  'Approved By Email': User().Email, 'Approved At': Now()}));
If(!IsBlank(varSel.'Payment Reference'),
  Patch('MCHF - Payment Log', LookUp('MCHF - Payment Log', Reference = varSel.'Payment Reference'),
    {Status: {Value: "Coded"}}));
IfError('MCHF-ApproveGift'.Run(varSel.ID),
  Notify("The approval flow did not start. Check Historique.", NotificationType.Error),
  Notify("Approved. Recording in NXT… Reopen the app in about 2 minutes to see the result.", NotificationType.Success))
```

`btnCancel.OnSelect`:

```
If(IsBlank(txtComment.Text),
  Notify("Write why the submission is cancelled.", NotificationType.Error),
  Patch('MCHF - Submission History', Defaults('MCHF - Submission History'), {Title: "Cancel",
    'Submission Reference': varSel.Title, 'From Status': varSel.'Processing status'.Value, 'To Status': "Cancelled",
    Comment: txtComment.Text, 'Actor Email': User().Email, 'Action At': Now()});
  Set(varSel, Patch('MCHF Submissions', varSel, {'Processing status': {Value: "Cancelled"},
    'Review Comment': txtComment.Text}));
  If(!IsBlank(varSel.'Payment Reference'),
    Patch('MCHF - Payment Log', LookUp('MCHF - Payment Log', Reference = varSel.'Payment Reference'),
      {Status: {Value: "Unclaimed"}, 'Claimed By Name': "", 'Submission Reference': ""}));
  Reset(txtComment);
  Notify("Cancelled.", NotificationType.Success))
```

`btnToReturned` (on `scrReview`): Text `"My returned submissions"`, X 1140,
Y 690, Width 180, Height 40, OnSelect `Navigate(scrReturned)`.

### Fundraiser screen: scrReturned (T5)

The fundraiser sees their own submissions in Needs Information, reads the
comment of Gift Processing, fixes the amount, campaign or appeal, and
resubmits. The coding form of #8 is not changed.

| Control | Settings |
|---|---|
| Gallery `galReturned` | X 40, Y 40, Width 560, Height 690. Items `Filter('MCHF Submissions', 'Processing status'.Value = "Needs Information" && Author.Email = User().Email)` |
| Label `Title2` (in galReturned) | Text `ThisItem.Title` |
| Label `Subtitle2` (in galReturned) | Text `ThisItem.'Constituent Name' & " · " & Text(ThisItem.'Gift Amount') & " $"` |
| Label `lblRetComment` | X 640, Y 40, Width 680, Height 120, Color `RGBA(164, 38, 44, 1)`. Text below |
| Labels `lblRetAmount`, `lblRetCampaign`, `lblRetAppeal` | X 640, Y 180 / 240 / 300, Width 200, Height 40. Text `"Gift amount"`, `"Campaign"`, `"Appeal"` |
| Text input `txtRetAmount` | X 860, Y 180, Width 300, Height 40. Default `If(IsBlank(galReturned.Selected), "", Text(galReturned.Selected.'Gift Amount', "0.00"))` |
| Text inputs `txtRetCampaign`, `txtRetAppeal` | X 860, Y 240 / 300, Width 300, Height 40. Default `galReturned.Selected.Campaign` / `galReturned.Selected.Appeal` |
| Button `btnResubmit` | Text `"Resubmit"`, X 640, Y 380, Width 160, Height 40. OnSelect below |
| Button `btnToReview` | Text `"Back to review"`, X 1140, Y 690, Width 180, Height 40. OnSelect `Navigate(scrReview)` |

Every control of the screen reads `galReturned.Selected` (no variable). In
the lab, `Filter` takes a single condition: join conditions with `&&`.

`lblRetComment.Text`:

```
If(IsBlank(galReturned.Selected), "Select a returned submission.",
  galReturned.Selected.Title & " was returned by Gift Processing:" & Char(10) & galReturned.Selected.'Review Comment')
```

`btnResubmit.OnSelect`. It checks the fields, reads the submission again to
make sure it is still in Needs Information, refuses an amount change on a
gift split across several funds and an amount below the fees, then saves,
updates the single fund line, and adds one history row with the old and new
values:

```
With({amt: Trim(txtRetAmount.Text)},
  If(IsBlank(galReturned.Selected),
      Notify("Select a returned submission.", NotificationType.Error),
    !IsMatch(amt, "^\d+(\.\d{1,2})?$") || Value(amt) <= 0,
      Notify("Gift amount: a number more than 0, for example 100.00.", NotificationType.Error),
    IsBlank(Trim(txtRetCampaign.Text)) || IsBlank(Trim(txtRetAppeal.Text)),
      Notify("Campaign and appeal are required.", NotificationType.Error),
    Refresh('MCHF Submissions'); Refresh('MCHF Fund Lines');
    With({cur: LookUp('MCHF Submissions', ID = galReturned.Selected.ID),
          lines: Filter('MCHF Fund Lines', 'Submission Reference' = galReturned.Selected.Title)},
      If(cur.'Processing status'.Value <> "Needs Information",
          Notify(cur.Title & " is now " & cur.'Processing status'.Value & ". Nothing was changed.", NotificationType.Error),
        Value(amt) <> cur.'Gift Amount' && CountRows(lines) > 1,
          Notify("This gift is split across several funds. Ask Gift Processing before changing the amount.", NotificationType.Error),
        Value(amt) < Coalesce(cur.Fees, 0),
          Notify("The gift amount can't be less than the fees (" & Text(cur.Fees) & ").", NotificationType.Error),
        Patch('MCHF Submissions', cur, {'Gift Amount': Value(amt),
          'Net Amount': Value(amt) - Coalesce(cur.Fees, 0),
          Campaign: Trim(txtRetCampaign.Text), Appeal: Trim(txtRetAppeal.Text),
          'Processing status': {Value: "Submitted"}});
        If(CountRows(lines) = 1, Patch('MCHF Fund Lines', First(lines), {Amount: Value(amt)}));
        Patch('MCHF - Submission History', Defaults('MCHF - Submission History'), {Title: "Resubmit",
          'Submission Reference': cur.Title, 'From Status': "Needs Information", 'To Status': "Submitted",
          Comment: "Amount " & Text(cur.'Gift Amount') & " → " & amt & "; Campaign " & cur.Campaign & " → " & Trim(txtRetCampaign.Text) & "; Appeal " & cur.Appeal & " → " & Trim(txtRetAppeal.Text),
          'Actor Email': User().Email, 'Action At': Now()});
        Refresh('MCHF Submissions');
        Notify(cur.Title & " was resubmitted.", NotificationType.Success)))))
```

## 5. Differences with the Dev 4 guide

| Guide | What was built | Why |
|---|---|---|
| Child flow `MCHF - Record approved gift` | `MCHF - Record gift in NXT` | Name of the flow of #13 |
| Filter `SubmissionReference eq …` | `Submission_x0020_Reference eq …` | Internal name in `MCHF Fund Lines` |
| `LookUp('MCHF - Payment Log', Title = …)` | `Reference = …` | The `PAY-…` reference is in `Reference` (#3) |
| Reject clears `'Claimed By Email'` | Not cleared | The Payment Log has no such column |
| `Set(varSel, ThisItem)` on the gallery | `Set(varSel, galSubs.Selected)` | `ThisItem` isn't recognized there in the lab |
| Run a Child Flow "added later" | Available | Used here and in test 14 of #13 |
| Issue: a "fix a returned submission" mode in the coding form of #8 | The screen `scrReturned` in `MCHF - Review` | The coding form stays as #8 left it. The guide also allows a simple fundraiser screen |
| Issue: "the flow sets the status to Recording in Progress" | The app sets it, before calling the flow | As in the guide and `review-audit.md` (T7): the status is right even before the flow starts |

## 6. Tests

Test data: fictional donor Camille Exemple (`1234567`, system ID `11001`),
$100, Personal Cheque, `AC-2026-2027` / `MG-I-2026` / `HA-CARD0110`, one
fund line of $100 with the same Submission Reference.

| # | Test | Result |
|---|---|---|
| 1 | Flow run with Tester on `SUB-TEST15` | Passed: PDF in `MCHF - Audit documents`, opens, shows `SUB-TEST15`, one fund line and a $100 total. Link in Audit PDF. Added to NXT Batch, history row with the detail of #13 (batch 13) |
| 2 | Flow run with Tester on `SUB-TEST15F` (`9999001`, system ID `11002`) | Passed: Transfer Failed, never Added to NXT Batch. History: "Add a gift to batch failed (HTTP 500) … Nothing was added to batch 14." |
| 3 | Claim in the app (`SUB-TEST15A`) | Passed: Under Review, Reviewer Email, history row |
| 4 | Approve in the app (`SUB-TEST15A`, `PAY-00016`) | Passed: Recording in Progress, then Added to NXT Batch after about 2 minutes. `PAY-00016` is Coded. The PDF shows the payment reference, Approved by and Approved at. The flow keeps running after it answers the app |
| 5 | Approve with an empty Constituent System ID (first try of test 4) | Passed: Transfer Failed, "Not sent to NXT: Constituent System ID is empty" |
| 6 | Reject without a comment (`SUB-TEST15R`) | Passed: error message, status unchanged |
| 7 | Reject with a comment (`SUB-TEST15R`, `PAY-00017`) | Passed: Rejected, Review Comment saved, no buttons left. `PAY-00017` is Unclaimed, Claimed By Name and Submission Reference empty |
| 8 | Return with a comment (`SUB-TEST15N`) | Passed: Needs Information, Review Comment saved, only Cancel left |
| 9 | My returned submissions, `SUB-TEST15N`, Appeal `MG-I-2026` → `MG-I-2026-B`, Resubmit | Passed: the comment shows in red, Submitted, the submission leaves the list, history row "Resubmit" with "Appeal MG-I-2026 → MG-I-2026-B" |
| 10 | Claim, then Return, on a new-donor submission without Constituent ID (`SUB-TEST15D`) | Passed after the list was added again in Données (section 7) |
| 11 | My returned submissions | Passed: only the returned submissions created by the user (`SUB-TEST15D`), not `SUB-TEST15N` once resubmitted |
| 12 | Resubmit `SUB-TEST15D` (fees $5) with a gift amount of 3 | Passed: "The gift amount can't be less than the fees (5).", nothing saved |
| 13 | Cancel `SUB-TEST15D` from Needs Information (`PAY-00017`) | Passed: Cancelled, `PAY-00017` is Unclaimed, Claimed By Name and Submission Reference empty |

## 7. Limits

- The review screen doesn't refresh by itself. Reopen the app about 2
  minutes after Approve to see the result of the flow. `Refresh` now exists
  in the lab (used by Resubmit); the review buttons get it in #27.
- Resubmit can change only the amount, the campaign and the appeal. Its
  history comment lists the three fields even when one didn't change.
- The fundraiser screen shows the submissions created by the person using
  the app (`Author`). Security by role comes in Sprint 4.
- An app keeps the columns of a list as they were when the list was added.
  After `Constituent ID` became optional (#8), `MCHF Submissions` had to be
  removed and added again in Données, otherwise a submission without
  Constituent ID could not be claimed. Do the same after any column change.
- A comment made only of spaces is accepted by Return, Reject and Cancel.
  #27 adds `Trim` and the status check to every review button.
- If a card of the PDF chain fails, the flow stops and the submission stays
  in Recording in Progress. Fix the card, then **Resoumettre** the run in
  Historique.
- Fund lines are found by `Submission_x0020_Reference`. A fund line saved
  without it is missing from the PDF (empty table, total 0), and #13 then
  sends the whole amount on the primary fund. The coding form (#8) must
  always fill it.
- `utcNow()` and `Now()` give UTC in the file name and in the PDF.
- Once, after Approve, the lab showed "La session de l'application est
  fermée (service Power Fx injoignable ?)". The flow still finished. To
  report to Jérémie.
- Security by role (who is Gift Processing) is not handled. Planned for
  Sprint 4.
- The buttons of Transfer Failed (T11, T12) are not built yet.

## Deliverables

- Rebuild document: `docs/rebuild/review-app-approval.md`
- The app `MCHF - Review` and the flow `MCHF - ApproveGift` in Équipe
- Six columns of `MCHF Submissions` added to `data-dictionary.md`
- Lab export of the flow, the app, `MCHF Submissions` and `MCHF - Audit
  documents` (no list items): `lab/solutions/mchf-review-approval.solution.json`.
  Importing it reuses lists with the same name, and the flow comes in
  turned off. The library comes without its files.

## Remaining work

- The Transfer Failed buttons T11 and T12, and the status check in every
  review button (#27).
