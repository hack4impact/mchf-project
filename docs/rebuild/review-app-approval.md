# Review App, Approval Flow and Simulated Transfer

Related issues: [#15](https://github.com/hack4impact/mchf-project/issues/15), [#27](https://github.com/hack4impact/mchf-project/issues/27), [#28](https://github.com/hack4impact/mchf-project/issues/28), [#34](https://github.com/hack4impact/mchf-project/issues/34), [#51](https://github.com/hack4impact/mchf-project/issues/51)
Branches: `feature/15-correction-loop-and-transfer`, `feature/27-review-screen-actions`, `feature/28-review-correct`, `feature/34-audit-pdf-sprint3-fields`, `feature/51-review-history`
Environment: Équipe

Gift Processing reviews a submission in the app **MCHF - Review**: claim,
return with a comment, reject, approve or cancel, correct a submission
under review without sending it back, and after a Transfer Failed checks
NXT before recording again. Approve calls the flow
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
| `'MCHF Fund Lines'` | Read by the flow for the PDF, and shown in the detail (#28) |
| `'MCHF - Submission History'` | One row per action (#4, section 7) |
| `'MCHF - Payment Log'` | The linked payment. Its `PAY-…` reference is in the column `Reference`, not `Title` |
| `'MCHF-ApproveGift'` | The approval flow, called with `.Run` |
| `'MCHF Payment Types'` | Choices of the Correct mode (#28) |
| `RENXT` | Campaigns, appeals and the donor lookup of the Correct mode (#28) |

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
| PDF Needed | Condition | After the loop. `{"and": [{"equals": ["@empty(body('Get_submission')?['AuditPDF'])", true]}]}`. The 7 cards below are in its yes branch; the no branch is empty (#27) |
| HTML (in PDF Needed, yes) | Compose | The template of #11 with every placeholder replaced (section 3) |
| Create HTML file (in PDF Needed, yes) | OneDrive for Business, Create file | Folder `/MCHF-Audit`, name `@{body('Get_submission')?['Title']}-@{formatDateTime(utcNow(), 'yyyyMMdd-HHmmss')}.html`, content `@outputs('HTML')` |
| Wait (in PDF Needed, yes) | Delay | 1 minute |
| To PDF (in PDF Needed, yes) | OneDrive for Business, Convert file | File `@outputs('Create_HTML_file')?['body/Id']`, target type PDF |
| Create PDF file (in PDF Needed, yes) | SharePoint, Create file | Site: environment of the flow, folder `/MCHF - Audit documents`, name `@{outputs('Create_HTML_file')?['body/NameNoExt']}.pdf`, content `@body('To_PDF')` |
| Get link (in PDF Needed, yes) | Get file properties | Library `MCHF - Audit documents`, Id `@outputs('Create_PDF_file')?['body/ItemId']` |
| Save link (in PDF Needed, yes) | Update item | `MCHF Submissions`, Id = `SubmissionID`, Audit PDF = `@{outputs('Get_link')?['body/{Link}']}`. Every other field empty |
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

When the submission already has an Audit PDF ("record again", T12), the
flow skips the PDF and goes straight to Record in NXT: the PDF keeps the
date of the approval.

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
<tr><td><strong>Constituent name</strong></td><td>@{replace(replace(replace(coalesce(body('Get_submission')?['ConstituentName'], ''), '&', '&amp;'), '<', '&lt;'), '>', '&gt;')}</td></tr>
<tr><td><strong>Constituent system ID</strong></td><td>@{body('Get_submission')?['Constituent_x0020_System_x0020_I']}</td></tr>
<tr><td><strong>New donor</strong></td><td>@{if(equals(body('Get_submission')?['NewDonor'], true), 'Yes', 'No')}</td></tr>
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

<h2>Tax receipt</h2>
<table>
<tr><td><strong>Tax receipt to issue</strong></td><td>@{if(equals(body('Get_submission')?['TaxReceipt'], true), 'Yes', 'No')}</td></tr>
<tr><td><strong>Benefit amount</strong></td><td>@{body('Get_submission')?['BenefitAmount']}</td></tr>
<tr><td><strong>Receipt amount</strong></td><td>@{body('Get_submission')?['ReceiptAmount']}</td></tr>
</table>

<h2>Notes</h2>
<p>@{replace(replace(replace(coalesce(body('Get_submission')?['Notes'], ''), '&', '&amp;'), '<', '&lt;'), '>', '&gt;')}</p>

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

The Sprint 3 fields of this card come from #34. They are in a personal copy
of the flow until the change request is merged in Équipe. The exports in
`lab/solutions` don't have them yet (`audit-pdf.md`, Remaining work).

## 4. Review app: MCHF - Review

Applications › Nouvelle application in Équipe: `MCHF - Review`, tablet,
formulas in English. Données: the lists of section 1, the flow and the
RENXT connector.

The app is in the lab export (Deliverables), but it can't be exported to
the real Power Apps yet: rebuild it from this section.

### Controls

| Control | Settings |
|---|---|
| Screen `scrReview` | |
| Gallery `galSubs` | X 40, Y 40, Width 560, Height 690. Items `Sort('MCHF Submissions', ID, SortOrder.Descending)`. OnSelect `Set(varSel, galSubs.Selected); Set(varEditing, false)` (choosing another submission closes the Correct mode, #28) |
| Label `Title1` (in galSubs) | Text `ThisItem.Title` |
| Label `Subtitle1` (in galSubs) | Text `ThisItem.'Constituent Name' & " · " & Text(ThisItem.'Gift Amount') & " $ · " & ThisItem.'Processing status'.Value` |
| Label `lblDetail` | X 640, Y 40, Width 680, Height 270, VerticalAlign `VerticalAlign.Top`. Text below |
| Gallery `galFundLines`, label `lblFundCheck` | Y 315 and Y 390 (#28, section below) |
| Text input `txtComment` | X 640, Y 420, Width 680, Height 60. Mode `TextMode.MultiLine`, Default `""`, HintText `"Comment for the fundraiser (required for Return and Reject)"` |
| Buttons | Y 495, Width 120, Height 40. Claim and Return at X 640 (never visible together), Reject X 780, Approve X 920, Cancel X 1060, Correct X 1200. Transfer Failed: `btnInNxt` X 640 and `btnRecordAgain` X 880 (section below) |
| Label `lblCheckNxt` | X 640, Y 540, Width 680, Height 120. Only in Transfer Failed (section below) |
| Buttons `btnRefresh`, `btnToReturned` | Y 690, Width 180, Height 40, X 940 and X 1140 |
| Correct mode controls, `lblNeedsId` | Y 550 to 738 (#28, section below) |

In the lab, `ThisItem` isn't recognized in the OnSelect of the gallery
itself, only in the controls of its template. The gallery uses
`galSubs.Selected`.

`lblDetail.Text` reads the submission again with `LookUp` instead of using
`varSel` (#28, section 7):

```
If(IsBlank(varSel), "Select a submission.",
 With({s: LookUp('MCHF Submissions', ID = varSel.ID)},
  "Reference: " & s.Title & Char(10) &
  "Donor: " & s.'Constituent Name' & " (" & s.'Constituent ID' & ")" & Char(10) &
  "New donor: " & If(s.'New Donor', "Yes", "No") & Char(10) &
  "Gift: " & Text(s.'Gift Amount') & " $, " & s.'Payment Type' & Char(10) &
  "Coding: " & s.Campaign & " / " & s.Appeal & " / " & s.Fund & Char(10) &
  "Package: " & s.Package & Char(10) &
  "Assigned solicitor: " & s.'Assigned Solicitor' & Char(10) &
  "Tax receipt: " & If(s.'Tax Receipt', "Yes", "No") &
    "   Receipt: " & If(IsBlank(s.'Receipt Amount'), "", Text(s.'Receipt Amount', "0.00")) &
    "   Benefit: " & If(IsBlank(s.'Benefit Amount'), "", Text(s.'Benefit Amount', "0.00")) & Char(10) &
  "Notes: " & s.Notes & Char(10) &
  "Payment reference: " & s.'Payment Reference' & Char(10) &
  "Status: " & s.'Processing status'.Value & Char(10) &
  "Review comment: " & s.'Review Comment'))
```

### Buttons

Each button is visible only in its From statuses (`review-audit.md`,
section 4). Each one reads the submission again just before writing
(`Refresh`, then `LookUp`), refuses if the status changed in the meantime,
then writes one history row, updates the submission, and shows a
notification (#27, Sprint 3 guide section 7).

| Button | Visible | Transition |
|---|---|---|
| `btnClaim` | `varSel.'Processing status'.Value = "Submitted"` | T2 |
| `btnReturn` | `varSel.'Processing status'.Value = "Under Review"` | T4 |
| `btnReject` | `varSel.'Processing status'.Value = "Under Review"` | T6 |
| `btnCorrect` | `varSel.'Processing status'.Value = "Under Review" && !varEditing` | T3 (#28) |
| `btnApprove` | `varSel.'Processing status'.Value = "Under Review"` | T7 |
| `btnCancel` | `varSel.'Processing status'.Value in ["Submitted", "Under Review", "Needs Information"]` | T13 |
| `btnInNxt` | `varSel.'Processing status'.Value = "Transfer Failed"` | T11 |
| `btnRecordAgain` | `varSel.'Processing status'.Value = "Transfer Failed"` | T12 |

No status-changing review button shows in Recording in Progress, Added to NXT Batch, Rejected or
Cancelled. There is no Retry: a gift is never sent again before someone
checked NXT.

`btnRefresh` (Text `"Actualiser"`, X 940, Y 690, Width 180, Height 40):

```
Refresh('MCHF Submissions'); Refresh('MCHF - Submission History');
If(!IsBlank(varSel), Set(varSel, LookUp('MCHF Submissions', ID = varSel.ID)))
```

`btnClaim.OnSelect`:

```
Refresh('MCHF Submissions');
With({cur: LookUp('MCHF Submissions', ID = varSel.ID)},
    If(cur.'Processing status'.Value <> "Submitted",
        Notify(cur.Title & " is now " & cur.'Processing status'.Value & ". Nothing was changed.", NotificationType.Error);
        Set(varSel, cur),
        Set(varSel, Patch('MCHF Submissions', cur, {'Processing status': {Value: "Under Review"}, 'Reviewer Email': User().Email}));
        Patch('MCHF - Submission History', Defaults('MCHF - Submission History'), {Title: "Claim",
            'Submission Reference': cur.Title, 'From Status': cur.'Processing status'.Value, 'To Status': "Under Review",
            'Actor Email': User().Email, 'Action At': Now()});
        Notify("Claimed. You are the reviewer of " & cur.Title & ".", NotificationType.Success)))
```

`btnReturn.OnSelect`. A comment made only of spaces is refused (`Trim`):

```
If(IsBlank(Trim(txtComment.Text)), Notify("Write a comment for the fundraiser.", NotificationType.Error),
    Refresh('MCHF Submissions');
    With({cur: LookUp('MCHF Submissions', ID = varSel.ID)},
        If(cur.'Processing status'.Value <> "Under Review",
            Notify(cur.Title & " is now " & cur.'Processing status'.Value & ". Nothing was changed.", NotificationType.Error);
            Set(varSel, cur),
            Set(varSel, Patch('MCHF Submissions', cur, {'Processing status': {Value: "Needs Information"}, 'Review Comment': txtComment.Text}));
            Patch('MCHF - Submission History', Defaults('MCHF - Submission History'), {Title: "Return",
                'Submission Reference': cur.Title, 'From Status': cur.'Processing status'.Value, 'To Status': "Needs Information",
                Comment: txtComment.Text, 'Actor Email': User().Email, 'Action At': Now()});
            Reset(txtComment);
            Notify(cur.Title & " returned to the fundraiser.", NotificationType.Success))))
```

`btnReject.OnSelect`. The linked payment goes back to Unclaimed, only when
it is found (#3):

```
If(IsBlank(Trim(txtComment.Text)), Notify("Write the reason for the fundraiser.", NotificationType.Error),
    Refresh('MCHF Submissions'); Refresh('MCHF - Payment Log');
    With({cur: LookUp('MCHF Submissions', ID = varSel.ID)},
        If(cur.'Processing status'.Value <> "Under Review",
            Notify(cur.Title & " is now " & cur.'Processing status'.Value & ". Nothing was changed.", NotificationType.Error);
            Set(varSel, cur),
            Set(varSel, Patch('MCHF Submissions', cur, {'Processing status': {Value: "Rejected"}, 'Review Comment': txtComment.Text}));
            Patch('MCHF - Submission History', Defaults('MCHF - Submission History'), {Title: "Reject",
                'Submission Reference': cur.Title, 'From Status': cur.'Processing status'.Value, 'To Status': "Rejected",
                Comment: txtComment.Text, 'Actor Email': User().Email, 'Action At': Now()});
            If(!IsBlank(cur.'Payment Reference'),
                With({pay: LookUp('MCHF - Payment Log', Reference = cur.'Payment Reference')},
                    If(!IsBlank(pay), Patch('MCHF - Payment Log', pay,
                        {Status: {Value: "Unclaimed"}, 'Claimed By Name': Blank(), 'Submission Reference': Blank()}))));
            Reset(txtComment); Notify(cur.Title & " rejected.", NotificationType.Success))))
```

`btnApprove.OnSelect`. Approved by and Approved at are saved before the
flow runs, so the PDF shows them. The linked payment becomes Coded. A
submission without Constituent ID or Constituent System ID is refused
(#28), because the flow of #13 sends the System ID to NXT:

```
Refresh('MCHF Submissions'); Refresh('MCHF - Payment Log');
With({cur: LookUp('MCHF Submissions', ID = varSel.ID)},
    If(cur.'Processing status'.Value <> "Under Review",
        Notify(cur.Title & " is now " & cur.'Processing status'.Value & ". Nothing was changed.", NotificationType.Error);
        Set(varSel, cur),
       IsBlank(cur.'Constituent ID') || IsBlank(cur.'Constituent System ID'),
        Set(varSel, cur);
        Notify("Add the Constituent ID with Correct before approving.", NotificationType.Error),
        Set(varSel, Patch('MCHF Submissions', cur, {'Processing status': {Value: "Recording in Progress"},
            'Approved By Email': User().Email, 'Approved At': Now()}));
        Patch('MCHF - Submission History', Defaults('MCHF - Submission History'), {Title: "Approve",
            'Submission Reference': cur.Title, 'From Status': cur.'Processing status'.Value, 'To Status': "Recording in Progress",
            'Actor Email': User().Email, 'Action At': Now()});
        If(!IsBlank(cur.'Payment Reference'),
            With({pay: LookUp('MCHF - Payment Log', Reference = cur.'Payment Reference')},
                If(!IsBlank(pay), Patch('MCHF - Payment Log', pay, {Status: {Value: "Coded"}}))));
        IfError('MCHF-ApproveGift'.Run(cur.ID),
            Notify("The approval flow did not start. Check Historique.", NotificationType.Error),
            Notify(cur.Title & " approved. Recording in NXT… Click Actualiser in about 2 minutes.", NotificationType.Success))))
```

`btnCancel.OnSelect`:

```
If(IsBlank(Trim(txtComment.Text)), Notify("Write why the submission is cancelled.", NotificationType.Error),
    Refresh('MCHF Submissions'); Refresh('MCHF - Payment Log');
    With({cur: LookUp('MCHF Submissions', ID = varSel.ID)},
        If(!(cur.'Processing status'.Value in ["Submitted", "Under Review", "Needs Information"]),
            Notify(cur.Title & " is now " & cur.'Processing status'.Value & ". Nothing was changed.", NotificationType.Error);
            Set(varSel, cur),
            Set(varSel, Patch('MCHF Submissions', cur, {'Processing status': {Value: "Cancelled"}, 'Review Comment': txtComment.Text}));
            Patch('MCHF - Submission History', Defaults('MCHF - Submission History'), {Title: "Cancel",
                'Submission Reference': cur.Title, 'From Status': cur.'Processing status'.Value, 'To Status': "Cancelled",
                Comment: txtComment.Text, 'Actor Email': User().Email, 'Action At': Now()});
            If(!IsBlank(cur.'Payment Reference'),
                With({pay: LookUp('MCHF - Payment Log', Reference = cur.'Payment Reference')},
                    If(!IsBlank(pay), Patch('MCHF - Payment Log', pay,
                        {Status: {Value: "Unclaimed"}, 'Claimed By Name': Blank(), 'Submission Reference': Blank()}))));
            Reset(txtComment); Notify(cur.Title & " cancelled.", NotificationType.Success))))
```

### After a Transfer Failed (T11, T12)

Gift Processing first looks for the gift in NXT: the gift batch whose
description is the submission reference (in the lab: RE NXT (développeur) ›
Lots de dons › Voir les dons). Then it chooses one of the two buttons.

`lblCheckNxt` (X 640, Y 540, Width 680, Height 120, red, Visible
`varSel.'Processing status'.Value = "Transfer Failed"`):

```
"Check the NXT gift batches for " & varSel.Title & " before choosing. " &
First(Sort(Filter('MCHF - Submission History', 'Submission Reference' = varSel.Title), 'Action At', SortOrder.Descending)).Comment
```

`btnInNxt` (Text `"Gift is in NXT"`, X 640, Y 495, Width 220). Nothing is
sent to NXT. The comment is optional; the batch number is advised:

```
Refresh('MCHF Submissions');
With({cur: LookUp('MCHF Submissions', ID = varSel.ID)},
    If(cur.'Processing status'.Value <> "Transfer Failed",
        Notify(cur.Title & " is now " & cur.'Processing status'.Value & ". Nothing was changed.", NotificationType.Error);
        Set(varSel, cur),
        Set(varSel, Patch('MCHF Submissions', cur, {'Processing status': {Value: "Added to NXT Batch"}}));
        Patch('MCHF - Submission History', Defaults('MCHF - Submission History'), {Title: "Gift is in NXT",
            'Submission Reference': cur.Title, 'From Status': "Transfer Failed", 'To Status': "Added to NXT Batch",
            Comment: "Checked in NXT: " & txtComment.Text, 'Actor Email': User().Email, 'Action At': Now()});
        Reset(txtComment);
        Notify(cur.Title & " marked as added to the NXT batch. Nothing was sent again.", NotificationType.Success)))
```

`btnRecordAgain` (Text `"Gift is not in NXT, record again"`, X 880, Y 495,
Width 300). It runs the approval flow again; the flow keeps the first PDF
(section 2):

```
Refresh('MCHF Submissions');
With({cur: LookUp('MCHF Submissions', ID = varSel.ID)},
    If(cur.'Processing status'.Value <> "Transfer Failed",
        Notify(cur.Title & " is now " & cur.'Processing status'.Value & ". Nothing was changed.", NotificationType.Error);
        Set(varSel, cur),
        Set(varSel, Patch('MCHF Submissions', cur, {'Processing status': {Value: "Recording in Progress"}}));
        Patch('MCHF - Submission History', Defaults('MCHF - Submission History'), {Title: "Gift is not in NXT, record again",
            'Submission Reference': cur.Title, 'From Status': "Transfer Failed", 'To Status': "Recording in Progress",
            Comment: txtComment.Text, 'Actor Email': User().Email, 'Action At': Now()});
        Reset(txtComment);
        IfError('MCHF-ApproveGift'.Run(cur.ID),
            Notify("The approval flow did not start. Check Historique.", NotificationType.Error),
            Notify(cur.Title & ": recording again. Click Actualiser in about 2 minutes.", NotificationType.Success))))
```

`btnToReturned` (on `scrReview`): Text `"My returned submissions"`, X 1140,
Y 690, Width 180, Height 40, OnSelect `Navigate(scrReturned)`.

### Sprint 3 fields and Correct mode (#28)

**Fund lines in the detail.** Gallery `galFundLines` (X 640, Y 315, Width
680, Height 70, TemplateSize 22), Items
`Filter('MCHF Fund Lines', 'Submission Reference' = varSel.Title)`. Its
label `Title3` (Y 0, Height 22, Size 12): `ThisItem.Title & " : " & Text(ThisItem.Amount, "0.00") & " $"`.

`lblFundCheck` (X 640, Y 390, Width 680, red, Text `"Fund lines don't add
up to the gift amount"`). `Sum` of no lines gives blank in the lab, so it is
turned into 0:

```
!IsBlank(varSel) && Coalesce(Sum(Filter('MCHF Fund Lines', 'Submission Reference' = varSel.Title), Amount), 0) <> varSel.'Gift Amount'
```

**Choices.** `scrReview.OnVisible` loads the same choices as the coding
form (#8), without the empty first choice:

```
Set(varEditing, false);
ClearCollect(colPaymentTypes, ForAll(Sort(Filter('MCHF Payment Types', Active), 'Sort Order'), {Value: Title, code: Title}));
IfError(
    ClearCollect(colCampaigns, ForAll(RENXT.ListCampaigns().value, {Value: ThisRecord.lookup_id & " - " & ThisRecord.description, code: ThisRecord.lookup_id}));
    ClearCollect(colAppeals, ForAll(RENXT.ListAppeals().value, {Value: ThisRecord.lookup_id & " - " & ThisRecord.description, code: ThisRecord.lookup_id})),
    Notify("NXT lists are unavailable. Reopen the app to retry.", NotificationType.Error))
```

**btnCorrect** (Text `"Correct"`, X 1200, Y 495, Width 120, Height 40).
It reads the submission again, resets the fields so nothing typed before
a Discard comes back, then opens the Correct mode:

```
Refresh('MCHF Submissions');
With({cur: LookUp('MCHF Submissions', ID = varSel.ID)},
    Set(varSel, cur);
    If(cur.'Processing status'.Value <> "Under Review",
        Notify(cur.Title & " is now " & cur.'Processing status'.Value & ". Nothing was changed.", NotificationType.Error),
        Set(varEditState, ""); Set(varEditLookupId, ""); Set(varEditSystemId, ""); Set(varEditName, ""); Set(varEditMessage, "");
        Reset(txtEditConstituentId); Reset(cbxEditPaymentType); Reset(dpEditGiftDate); Reset(cbxEditCampaign);
        Reset(cbxEditAppeal); Reset(txtEditPackage); Reset(txtEditNotes);
        Set(varEditing, true)))
```

**Fields of the Correct mode.** All have Visible `varEditing`. Their
default reads `LookUp('MCHF Submissions', ID = varSel.ID)`, written `cur`
below.

| Control | Position (X, Y, Width) | Default |
|---|---|---|
| Text input `txtEditConstituentId` | 640, 550, 160 | `cur.'Constituent ID'`, HintText `"Constituent ID"` |
| Button `btnEditLookup` (`"Look up"`) | 810, 550, 100 | OnSelect below |
| Label `lblEditDonor` | 920, 550, 400 | `If(varEditState = "found", "Found: " & varEditName & " (" & varEditLookupId & ")", varEditMessage)`, green when found, red otherwise |
| Combo box `cbxEditPaymentType` | 640, 595, 160 | Items `colPaymentTypes`, DefaultSelectedItems `Filter(colPaymentTypes, code = cur.'Payment Type')` |
| Date picker `dpEditGiftDate` | 810, 595, 160 | DefaultDate `cur.'Gift Date'` |
| Combo box `cbxEditCampaign` | 980, 595, 165 | Items `colCampaigns`, DefaultSelectedItems `Filter(colCampaigns, code = cur.Campaign)` |
| Combo box `cbxEditAppeal` | 1155, 595, 165 | Items `colAppeals`, DefaultSelectedItems `Filter(colAppeals, code = cur.Appeal)` |
| Text input `txtEditPackage` | 640, 640, 160 | `cur.Package` |
| Text input `txtEditNotes` | 810, 640, 510 | `cur.Notes` |
| Button `btnSaveCorrection` (`"Save correction"`) | 640, 690, 160 | OnSelect below |
| Button `btnCancelCorrection` (`"Discard"`) | 810, 690, 120 | `Set(varEditing, false); Set(varEditState, ""); Set(varEditMessage, "")` |

The combo boxes have SelectMultiple `false`. Read-only in Correct: Gift
Amount and the fund lines (a new amount means Return to the fundraiser),
and this sprint the solicitor and the receipt amounts, which need the checks
of the coding form. New Donor stays Yes: it says where the submission came
from. The button is called Discard because Cancel already cancels the
submission.

`btnEditLookup.OnSelect` is the donor lookup of the coding form (#2, #8),
written to text variables:

```
With({typedId: Trim(txtEditConstituentId.Text)},
    If(!IsMatch(typedId, "\d{7}"),
        Set(varEditState, "invalid"); Set(varEditLookupId, ""); Set(varEditSystemId, ""); Set(varEditName, "");
        Set(varEditMessage, "Enter the 7-digit Constituent ID."),
        With({res: IfError(
            With({r: Filter(RENXT.SearchConstituents(typedId, {search_field: "lookup_id"}).value, lookup_id = typedId)},
                Switch(CountRows(r),
                    0, {state: "notfound", systemId: "", lookupId: "", name: "", message: "No donor with this Constituent ID. Check the number."},
                    1, {state: "found", systemId: Text(First(r).id), lookupId: First(r).lookup_id, name: First(r).name, message: ""},
                    {state: "error", systemId: "", lookupId: "", name: "", message: "Several donors match. Contact Gift Processing."})),
            {state: "error", systemId: "", lookupId: "", name: "", message: "Donor lookup unavailable. Try again in a moment."})},
            Set(varEditState, res.state); Set(varEditLookupId, res.lookupId); Set(varEditSystemId, res.systemId);
            Set(varEditName, res.name); Set(varEditMessage, res.message))))
```

`btnSaveCorrection.OnSelect`. It refuses if the status changed, or if the
Constituent ID was changed without a successful lookup. It writes only the
fields that changed, keeps the status, and adds one history row listing
each change as `Field: old → new`. A combo box left empty keeps the old
value:

```
Refresh('MCHF Submissions');
With({cur: LookUp('MCHF Submissions', ID = varSel.ID)},
 With({donorChanged: varEditState = "found" && varEditLookupId = Trim(txtEditConstituentId.Text) && varEditLookupId <> Coalesce(cur.'Constituent ID', ""),
       newPay: Coalesce(cbxEditPaymentType.Selected.code, cur.'Payment Type'),
       newDate: Coalesce(dpEditGiftDate.SelectedDate, cur.'Gift Date'),
       newCamp: Coalesce(cbxEditCampaign.Selected.code, cur.Campaign),
       newApp: Coalesce(cbxEditAppeal.Selected.code, cur.Appeal),
       newPkg: Trim(txtEditPackage.Text),
       newNotes: txtEditNotes.Text},
  If(cur.'Processing status'.Value <> "Under Review",
       Set(varSel, cur); Set(varEditing, false);
       Notify(cur.Title & " is now " & cur.'Processing status'.Value & ". Nothing was changed.", NotificationType.Error),
     Trim(txtEditConstituentId.Text) <> Coalesce(cur.'Constituent ID', "") && !donorChanged,
       Notify("Look up the new Constituent ID before saving.", NotificationType.Error),
     With({changes: Concat(Filter(Table(
            {f: "Constituent ID", o: cur.'Constituent ID', n: If(donorChanged, varEditLookupId, cur.'Constituent ID')},
            {f: "Payment Type", o: cur.'Payment Type', n: newPay},
            {f: "Gift Date", o: Text(cur.'Gift Date', "yyyy-mm-dd"), n: Text(newDate, "yyyy-mm-dd")},
            {f: "Campaign", o: cur.Campaign, n: newCamp},
            {f: "Appeal", o: cur.Appeal, n: newApp},
            {f: "Package", o: cur.Package, n: newPkg},
            {f: "Notes", o: cur.Notes, n: newNotes}),
          Coalesce(o, "") <> Coalesce(n, "")),
          f & ": " & Coalesce(o, "(empty)") & " → " & Coalesce(n, "(empty)"), "; ")},
       If(IsBlank(changes),
          Notify("Nothing changed.", NotificationType.Warning),
          Set(varSel, Patch('MCHF Submissions', cur, {
              'Constituent ID': If(donorChanged, varEditLookupId, cur.'Constituent ID'),
              'Constituent Name': If(donorChanged, varEditName, cur.'Constituent Name'),
              'Constituent System ID': If(donorChanged, varEditSystemId, cur.'Constituent System ID'),
              'Payment Type': newPay, 'Gift Date': newDate, Campaign: newCamp, Appeal: newApp,
              Package: newPkg, Notes: newNotes}));
          Patch('MCHF - Submission History', Defaults('MCHF - Submission History'), {Title: "Correct",
              'Submission Reference': cur.Title, 'From Status': "Under Review", 'To Status': "Under Review",
              Comment: changes, 'Actor Email': User().Email, 'Action At': Now()});
          Set(varEditing, false);
          Notify("Correction saved.", NotificationType.Success))))))
```

**Approve without Constituent ID.** `btnApprove.DisplayMode`:

```
If(IsBlank(varSel.'Constituent ID') || IsBlank(varSel.'Constituent System ID'), DisplayMode.Disabled, DisplayMode.Edit)
```

`lblNeedsId` (X 640, Y 738, Width 680, Height 24, red, Text `"New donor:
create the constituent in NXT, then add its Constituent ID with Correct."`):

```
!IsBlank(varSel) && varSel.'Processing status'.Value = "Under Review" && (IsBlank(varSel.'Constituent ID') || IsBlank(varSel.'Constituent System ID'))
```

The disabled button isn't enough, because `varSel` can be old: the OnSelect
of Approve checks again (see `btnApprove.OnSelect` above).

### Activity history and audit PDF (#51)

On `scrReview`, place **Activity history** below **Review actions**, following the history panel in #32. The submission queue is on the left. The right side shows the selected gift, fundraising coding, fund lines, review actions and history. The positions in the earlier Sprint 3 instructions describe the previous layout. Use `lab/solutions/mchf-review-history.solution.json` for the #51 layout.

The selected submission is stored in `varSel`. Add a vertical gallery named `galHistory` with these properties:

| Property | Formula |
|---|---|
| Items | `Sort(Filter('MCHF - Submission History', 'Submission Reference' = varSel.Title), 'Action At', SortOrder.Descending)` |
| Visible | `!IsBlank(varSel.ID)` |
| Y | `If(varEditing, 675, 587)` |
| Height | `If(varEditing, 78, 166)` |
| TemplateSize | `77` |

Add these labels inside the gallery row:

| Label | Text |
|---|---|
| `lblHistoryAction` | `ThisItem.Title & " · " & If(IsBlank(ThisItem.'From Status'), "", ThisItem.'From Status' & " → ") & ThisItem.'To Status'` |
| `lblHistoryActor` | `ThisItem.'Actor Email' & " · " & Text(ThisItem.'Action At', "yyyy-mm-dd hh:mm")` |
| `lblHistoryComment` | `ThisItem.Comment` |

The gallery reads the comment saved by each action. A Correct row shows the field and its old and new values, for example `Notes: (empty) → History Test`. A recording result written by the approval flow displays `System` as the actor.

Place `btnOpenPdf` beside the history heading. Use the selected submission's saved link:

| Property | Formula |
|---|---|
| Text | `"Open audit PDF"` |
| Visible | `!IsBlank(LookUp('MCHF Submissions', ID = varSel.ID).'Audit PDF')` |
| OnSelect | `Launch(LookUp('MCHF Submissions', ID = varSel.ID).'Audit PDF')` |

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
| #28: `Status.Value` in the Correct formulas | `'Processing status'.Value` | The display name of the column in the app |
| #28: `varEditDonor` (a record) | Five text variables `varEditState`, `varEditLookupId`, `varEditSystemId`, `varEditName`, `varEditMessage` | The lab refused a record in `Set` ("Expecting a Blank value"). The coding form also uses text variables |
| #28: fields read `varSel` | `LookUp('MCHF Submissions', ID = varSel.ID)` | `varSel` kept the columns from before #21 (section 7) |
| #28: a cleared combo box writes blank | It keeps the old value (`Coalesce`) | A correction never erases a required code by mistake |
| #28: button "Cancel" for the Correct mode | "Discard" | Cancel already cancels the submission |
| #28: `lblNeedsId` visible whenever the ID is empty | Only in Under Review | The message talks about Approve, which only exists there |

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
| 14 | #27: Approve `SUB-TEST27C` (`9999003`, system ID `11004`) | Passed: PDF made, Transfer Failed, only "Gift is in NXT" and "Gift is not in NXT, record again", no Retry, `lblCheckNxt` shows the uncertain detail of #13 |
| 15 | #27: batch `SUB-TEST27C` in RE NXT (développeur) › Lots de dons, then Gift is in NXT with "Batch 1016, 1 gift of $100." | Passed: the batch holds 1 gift; Added to NXT Batch, no buttons, history "Checked in NXT: Batch 1016…", nothing sent again |
| 16 | #27: Approve `SUB-TEST27B` (`9999002`, system ID `11003`), empty batch, then record again | Passed: Transfer Failed twice (expected for `9999002`). Audit PDF unchanged, the run of PDF Needed took the no branch, a second "Recording result" row |
| 17 | #27: Return, Reject and Cancel with a comment of 3 spaces (`SUB-TEST15E`) | Passed: the three are refused, status unchanged |
| 18 | #27: Cancel `SUB-TEST15E`, then filter the history on `SubmissionReference eq 'SUB-TEST15E'` | Passed: exactly one row; the refused clicks wrote nothing |
| 19 | #27: Reject `SUB-TEST15N` linked to `PAY-00017` (Claimed) | Passed: Rejected, `PAY-00017` Unclaimed, Claimed By Name and Submission Reference empty |
| 20 | #27: Claim from two tabs, and Claim after the status was changed in the list | The lab updates the open app right away: Claim disappears before it can be clicked, and only one Claim row is written. The refusal message could not be shown in the lab |
| 21 | #28: detail of `SUB-TEST15A` and `SUB-20261110-001004` (no fund line) | Passed: the Sprint 3 fields show. `HA-CARD0110 : 100.00 $` and no warning for `SUB-TEST15A`; "Fund lines don't add up to the gift amount" for `SUB-20261110-001004` |
| 22 | #28: `SUB-TEST28A` (Under Review), Correct, Package empty → `GALA-2026`, Save correction | Passed: status still Under Review, only Package changed, one history row "Correct" with "Package: (empty) → GALA-2026", fund lines still total $100 |
| 23 | #28: Correct, nothing changed, Save correction, then Discard | Passed: "Nothing changed.", no history row; Discard closes the fields |
| 24 | #28: `SUB-TEST28N` (New Donor Yes, no Constituent ID, Under Review) | Passed: "New donor: Yes", Approve disabled, `lblNeedsId` shown |
| 25 | #28: Correct, `1234567` typed, Save correction without Look up | Passed: "Look up the new Constituent ID before saving.", nothing saved |
| 26 | #28: Look up (`Found: Camille Exemple (1234567)`), Save correction | Passed: Constituent ID `1234567`, name and System ID `11001` saved, status Under Review, one history row "Constituent ID: (empty) → 1234567", Approve enabled, `lblNeedsId` hidden, fund lines still total $100. Approve not clicked |
| 27 | #28: `SUB-TEST28A`, Correct, Package `XXX`, Discard, Correct again | Passed: Package shows `GALA-2026` again, nothing saved |
| 28 | #28: Correct with an expired RE NXT connection | Campaign and Appeal empty and Look up fails (401). Connexions › Reconnecter fixes it (section 7) |

## 7. Limits

- After Approve or record again, click Actualiser about 2 minutes later to
  see the result of the flow.
- Each button reads the submission again before writing. In the lab the
  open app already follows changes made elsewhere, so the "is now …" refusal
  could not be shown (test 20). The check stays for the real Power Apps,
  where data is cached.
- Resubmit can change only the amount, the campaign and the appeal. Its
  history comment lists the three fields even when one didn't change.
- The fundraiser screen shows the submissions created by the person using
  the app (`Author`). Security by role comes in Sprint 4.
- An app keeps the columns of a list as they were when the list was added.
  After `Constituent ID` became optional (#8), `MCHF Submissions` had to be
  removed and added again in Données, otherwise a submission without
  Constituent ID could not be claimed. Do the same after any column change.
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
- #28: after `MCHF Submissions` was added again for the columns of #21, the
  list knew `'New Donor'` but `varSel.'New Donor'` stayed "isn't recognized":
  the editor keeps the old type of a variable. The new code reads the
  submission with `LookUp('MCHF Submissions', ID = varSel.ID)`. To report to
  Jérémie.
- #28: a multi-line formula pasted in the lab becomes one line, so a
  `// comment` at its start turns the whole formula into a comment, with no
  error. Don't start a pasted formula with a comment.
- #28: the Correct mode can't change the gift amount, the fund lines, the
  solicitor or the receipt amounts. A wrong amount is returned to the
  fundraiser.
- #28: the Correct mode calls RE NXT with the connection of the person using
  the app. With an expired connection, Campaign and Appeal are empty and the
  lookup fails (401): Connexions › Reconnecter. An empty combo box keeps the
  old value, so nothing is erased.

## Deliverables

- Rebuild document: `docs/rebuild/review-app-approval.md`
- The app `MCHF - Review` and the flow `MCHF - ApproveGift` in Équipe
- Six columns of `MCHF Submissions` added to `data-dictionary.md`
- Lab export of the flow, the app, `MCHF Submissions` and `MCHF - Audit
  documents` (no list items): `lab/solutions/mchf-review-approval.solution.json`.
  Importing it reuses lists with the same name, and the flow comes in
  turned off. The library comes without its files.
- #27: `lab/solutions/mchf-review-actions.solution.json` (the flow, the
  app and `MCHF Submissions`) and the flow plan
  `docs/rebuild/mchf-approve-gift-plan.pdf`.
- #28: `lab/solutions/mchf-review-correct.solution.json` (the app, the flow
  and the lists it uses). The flow did not change.
- #51: `lab/solutions/mchf-review-history.solution.json` (the review app).

## Remaining work

- Send a screenshot of the review screen to Hadyle (#32).
- #34: merge the change request of `MCHF - ApproveGift`, then export the
  flow again.
