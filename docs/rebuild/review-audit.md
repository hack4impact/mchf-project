# Review Actions, Processing Statuses, History and Teams Events

Related issue: [#4](https://github.com/hack4impact/mchf-project/issues/4)
Branch: `feature/4-review-actions-and-statuses`
Environment: Équipe

Defines how a submission moves through review: the actions, the
resulting statuses, the history kept and the Teams notifications.

Sections marked (proposal) still need to be confirmed by Marc and Yelene.

## 1. Two different claims

| Claim | Who | Record | Status change | Issue |
|---|---|---|---|---|
| Claim a payment | Fundraiser | `MCHF - Payment Log` | Unclaimed → Claimed | #3 |
| Claim a submission | Gift Processing | `MCHF Submissions` | Submitted → Under Review | #4 |

They affect separate records and separate status columns. This document
covers only the submission's Processing status.

## 2. Processing statuses

Statuses use the existing `Status` choice column on `MCHF Submissions`
(#1). No second status column is created.

| Status | Meaning | Final? |
|---|---|---|
| Submitted | Sent by the fundraiser, waiting for a reviewer | No |
| Under Review | A Gift Processing member claimed it and is reviewing it | No |
| Needs Information | Returned to the fundraiser with a comment | No |
| Rejected | Refused. The fundraiser starts a new submission | Yes |
| Cancelled | Sent in error, withdrawn | Yes |
| Recording in Progress | Approved, the recording flow (#13) is running | No |
| Added to NXT Batch | The gift is in an NXT gift batch | Yes |
| Transfer Failed | The recording failed or its result is uncertain | No |

Batch Committed is not used, because the solution cannot detect when Gift
Processing commits a batch in NXT.

## 3. Transitions table

| # | Action (button) | Who | From | To | Information kept |
|---|---|---|---|---|---|
| T1 | Submit (coding form, #8) | Fundraiser | *(new)* | Submitted | History row |
| T2 | Claim submission | Gift Processing | Submitted | Under Review | `ReviewerEmail` on the submission, history row |
| T3 | Correct | Gift Processing | Under Review | Under Review *(unchanged)* | History row, fields changed in Comment |
| T4 | Return with comment | Gift Processing | Under Review | Needs Information | Comment (required) in `ReviewComment`, visible to the fundraiser |
| T5 | Resubmit (#15) | Fundraiser | Needs Information | Submitted | History row |
| T6 | Reject | Gift Processing | Under Review | Rejected | Comment (required) in `ReviewComment` |
| T7 | Approve | Gift Processing | Under Review | Recording in Progress | `ApprovedByEmail` and `ApprovedAt` (also shown in the audit PDF, #11). Calls the flow MCHF - ApproveGift (#15) |
| T8 | *(recording flow result: success, #2)* | System | Recording in Progress | Added to NXT Batch | History row, Comment = detail from Dev 1 |
| T9 | *(recording flow result: failure, #2)* | System | Recording in Progress | Transfer Failed | History row, Comment = error detail from Dev 1 |
| T10 | *(recording flow result: uncertain, NXT timed out, #2)* | System | Recording in Progress | Transfer Failed | History row, Comment = "Check the batch in NXT before any retry" |
| T11 | Gift is in NXT | Gift Processing | Transfer Failed | Added to NXT Batch | Who checked, when. Nothing is sent again |
| T12 | Gift is not in NXT, record again | Gift Processing | Transfer Failed | Recording in Progress | Who checked, when. Restarts the recording flow |
| T13 | Cancel | See section 5 | Submitted, Needs Information, Under Review | Cancelled | Comment (required) in `ReviewComment` |

Any action not in this table is not allowed. A button is shown only when
the submission is in one of its From statuses.

### Rules

- Before saving, every button checks that the status is still one of its
  From statuses. If two reviewers click Claim at the same time, only the
  first one gets the submission.
- Return with comment is a status change from the review screen,
  not a Power Automate approval. Power Automate Approvals aren't used.
- Approve sets Recording in Progress in the app, then calls the flow
  MCHF - ApproveGift directly with `.Run`. The flow does not use a
  "when an item is modified" trigger, since it would start again every
  time the flow writes to the submission. The flow keeps the audit PDF
  (#11), then calls the recording flow of Dev 1 (#13) and sets Added to
  NXT Batch or Transfer Failed.
- Uncertain results are never retried automatically, since the gift may or
  may not be in NXT. The review screen shows the Comment of the latest
  history row so Gift Processing knows it must check NXT first.
- After Transfer Failed, Gift Processing first searches the NXT gift
  batches by the submission reference (`SUB-00042`), then chooses T11 or
  T12. Recording again (T12) is never possible without that check.
- A failed transfer never becomes Added to NXT Batch on its own.
  The only paths to Added to NXT Batch are a success from the recording
  flow (T8) or a person confirming the gift is in NXT (T11).
- Rejected and Cancelled are final. The submission is not
  reopened.

### Effect on the linked payment (#3)

When the submission has a `Payment Reference`:

| Submission action | Payment status |
|---|---|
| Approve (T7) | Coded |
| Reject (T6) or Cancel (T13) | Unclaimed (Claimed By Email, Claimed By Name and Submission Reference cleared) |

A failed transfer does not change the payment: it stays Coded, because
Coded means approved, not added to NXT (see #3).

## 4. Buttons by status (for the review screen, #12)

| Status | Buttons for Gift Processing | Buttons for the fundraiser |
|---|---|---|
| Submitted | Claim submission, Cancel | Cancel |
| Under Review | Correct, Return with comment, Reject, Approve, Cancel | None |
| Needs Information | Cancel | Resubmit (screen "My returned submissions" of MCHF - Review, #15), Cancel |
| Recording in Progress | None | None |
| Transfer Failed | "Gift is in NXT"; "Gift is not in NXT, record again" | None |
| Added to NXT Batch, Rejected, Cancelled | None | None |

## 5. Cancel (proposal)

| Who | Allowed while | Why |
|---|---|---|
| Fundraiser (their own submission) | Submitted, Needs Information | Nobody is reviewing it yet, or it is back in their hands |
| Gift Processing | Submitted, Under Review, Needs Information | Can withdraw a duplicate or a submission sent in error |
| Nobody | Recording in Progress and after | After Approve, the gift may already be in NXT |

To confirm with Yelene (through Marc).

## 6. Columns added to the submission list

Added to `MCHF Submissions` (#1). Yasmine is told before they are added.

| Display name | Internal name | Type | Use |
|---|---|---|---|
| Reviewer Email | `ReviewerEmail` | Single line of text | Gift Processing member who claimed the submission |
| Review Comment | `ReviewComment` | Multiple lines of text | Last comment from Return, Reject or Cancel. Shown to the fundraiser |
| Approved By Email | `ApprovedByEmail` | Single line of text | Who approved (audit PDF, #11) |
| Approved At | `ApprovedAt` | Date and time | When it was approved (audit PDF, #11) |
| Audit PDF | `AuditPDF` | Single line of text | Link to the audit PDF. The lab has no hyperlink column |
| Audit HTML | `AuditHTML` | Multiple lines of text | Fallback if the lab's file actions are not ready |

Each column is created with its internal name (no space), then renamed
to its display name.

Full history stays in the history list (section 7).

## 7. History list

The lab keeps no version history, so each action adds one row to its own
list.

List name: `MCHF - Submission History`

| Display name | Internal name | Type | Required | Content |
|---|---|---|---|---|
| Title | `Title` | Single line of text | Yes | Action name, e.g. `Approve` |
| Submission Reference | `SubmissionReference` | Single line of text | Yes | `SUB-00042` |
| From Status | `FromStatus` | Single line of text | No | Empty for Submit |
| To Status | `ToStatus` | Single line of text | Yes | Resulting status |
| Actor Email | `ActorEmail` | Single line of text | Yes | `User().Email`, or `System` for flow results |
| Comment | `Comment` | Multiple lines of text | No | Comment, fields changed, or detail from Dev 1 |
| Action At | `ActionAt` | Date and time | Yes | `Now()` |

This covers who submitted, every edit and the end of processing, each
with a timestamp.

### Rebuilding the history list

1. Select the Équipe environment.
2. Open Listes → Nouvelle liste.
3. Name the list MCHF - Submission History.
4. Add the columns using the internal names and types listed above.
   Create each one with its short name without spaces.
5. Rename the columns to their display names after creating them.
6. Enable the required setting for the five required fields.
7. Set Action At to date and time.

## 8. Teams events

Action: Post message in a chat or channel, as Flow bot. Plain text,
no adaptive cards or mentions.

| Event | When | Recipient | Message | Link |
|---|---|---|---|---|
| New submission | Status becomes Submitted (T1, T5) | Gift Processing channel | `New submission SUB-00042 from <fundraiser>, <amount> $.` For T5: `SUB-00042 was corrected and resubmitted by <fundraiser>.` | App MCHF - Review |
| Returned for information | Status becomes Needs Information (T4) | The fundraiser (chat) | `SUB-00042 was returned: <comment>. Open it to fix and resubmit.` | Coding form (#8) |
| Added to the batch | Status becomes Added to NXT Batch (T8, T11) | The fundraiser (chat) | `SUB-00042 was added to an NXT gift batch.` | None |

## 9. Open questions

1. Can a reviewer release a claim (Under Review → Submitted) to hand a
   submission to a colleague?
2. Should Transfer Failed also notify the Gift Processing channel?
3. Which Teams channel receives new submissions?

## Limits

- Security by role (who is Gift Processing) is not handled in the lab.
  It is planned for Sprint 4.
- If a step of the PDF chain fails, the flow stops and the submission
  stays in Recording in Progress. The error is fixed, then the run is
  replayed with Resoumettre in Historique.
- The app does not refresh by itself. It must be reopened to see the
  status set by the flow.

## Deliverables

- Rebuild document: `docs/rebuild/review-audit.md`
- Lab export of the history list:
  `lab/solutions/mchf-submission-history.solution.json`
- The six columns of section 6 are added to `MCHF Submissions` in Équipe.
  They appear in the next export of that list (#1).

## Remaining work

- Confirm the cancel proposal (section 5) with Marc and Yelene.
- Send the transitions table to Hadyle for the review screen.
- The transitions are documented rules. They are built in the review
  screen (#12) and the approval flow (#15). Test constituents from #2:
  `9999001` gives a failure, `9999002` and `9999003` an uncertain result.
