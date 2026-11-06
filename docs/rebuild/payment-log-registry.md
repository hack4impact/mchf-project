# Payment Log — Data and Rules

Related issue: [#3](https://github.com/hack4impact/mchf-project/issues/3)

## List

Name: MCHF - Payment Log
Environment: Équipe

Stores payments received by Gift Processing so fundraisers can claim
them and submit their coding information.

Only fictional data is used in the lab.

## Fields

| Internal name | Display name | Type | Required | Default |
|---|---|---|---|---|
| Title | Titre | Single line of text | Yes | None |
| ConstituentID | Constituent ID | Single line of text | Yes | None |
| ConstituentName | Constituent Name | Single line of text | Yes | None |
| PaymentType | Payment Type | Single line of text | Yes | None |
| GiftDate | Gift Date | Date only | Yes | None |
| GiftAmount | Gift Amount | Currency, minimum 0.01 | Yes | None |
| GiftType | Gift Type | Single line of text | No | None |
| AssignedSolicitor | Assigned Solicitor | Single line of text | No | None |
| Notes | Notes | Multiple lines of text | No | None |
| Reference | Reference | Single line of text | No | None |
| Status | Status | Choice | Yes | Unclaimed |
| ClaimedByName | Claimed By Name | Single line of text | No | None |
| SubmissionReference | Submission Reference | Single line of text | No | None |

Status choices are Unclaimed, Claimed and Coded.

The built-in Title column is retained.

Payment Type and Gift Type store text. The future payment-entry app
will obtain their dropdown options from Dev 2's configurable lists.

Gift Type is optional in the Payment Log, as required by this issue.
It is required in the Submission list.

## Rebuilding the list

1. Select the Équipe environment.
2. Open Listes → Nouvelle liste.
3. Name the list MCHF - Payment Log.
4. Add the columns using the internal names and types listed above.
5. Rename the columns to their display names after creating them.
6. Enable the required setting for the five required fields.
7. Set Gift Amount's minimum to 0.01.
8. Set Gift Date to date only.
9. Add the three Status choices and set Unclaimed as the default.

## Payment reference

The reference is built from the ID returned by the list after saving.

Format: PAY- followed by the ID padded to at least five digits.

Examples:
- ID 1 → PAY-00001
- ID 12 → PAY-00012

The first Patch creates the payment and stores the saved record in rec.
The second Patch updates that same record with its reference.

Reference is optional during the first save because the list must
assign an ID before the reference can be generated.

Once assigned, the reference must remain unchanged when the payment
is edited or its status changes.

### Reference test app

App name: MCHF - Payment Log Reference Test
Environment: Équipe
Data source: MCHF - Payment Log
Button: btnTestReference

To recreate the test, create a tablet app with English formulas,
add the Payment Log as a data source, and insert a button.

Set the button's OnSelect property to:

```powerfx
Set(
    rec,
    Patch(
        'MCHF - Payment Log',
        Defaults('MCHF - Payment Log'),
        {
            Title: "Reference test",
            'Constituent ID': "1234567",
            'Constituent Name': "Camille Exemple",
            'Payment Type': "Personal Cheque",
            'Gift Date': Today(),
            'Gift Amount': 100,
            Status: {Value: "Unclaimed"}
        }
    )
);
Patch(
    'MCHF - Payment Log',
    rec,
    {Reference: "PAY-" & Text(rec.ID, "00000")}
)
```

Save the app, open Aperçu and click the button once. Verify that the
new payment's Reference matches its list ID.

Each click creates a new fictional payment.

Payments added directly through the list do not automatically receive
a reference. Reference generation currently runs through this button.

## Status transitions

| Event | Resulting payment status | Associated action |
|---|---|---|
| Payment received and logged | Unclaimed | Assign the payment reference |
| Fundraiser claims the payment | Claimed | Store the fundraiser's name |
| Linked submission approved | Coded | Retain the linked submission reference |
| Linked submission rejected or cancelled | Unclaimed | Clear ClaimedByName and SubmissionReference so the payment can be claimed and linked to a new submission |

**Coded does not confirm a successful NXT transfer.**

Coded means Gift Processing approved the linked submission. Successful
addition to an NXT gift batch is tracked separately through the
submission's Added to NXT Batch processing status.

These transitions are documented rules. Their automatic execution
has not been implemented in this issue.

## Payment claims and submission claims

A fundraiser claims a payment in the registry:
- Record: Payment Log
- Status change: Unclaimed → Claimed
- Claimant stored in Claimed By Name

A Gift Processing member claims a submission for review:
- Record: Submission
- Processing status change: Submitted → Under Review

These actions affect separate records and separate status columns.

## Link to the submission

The payment's Reference identifies the payment, for example PAY-00002.

The Submission list's Payment reference column stores this PAY-…
reference when the submission originates from the registry. It remains
empty for submissions created independently.

The Payment Log's SubmissionReference column stores the reference
of the associated submission.

Submission references use the SUB-00001 format, as confirmed
in PR #18. SubmissionReference stores the reference of the
submission currently linked to the payment.

## Reminder rule

A reminder applies when a payment remains Unclaimed longer than
a configurable delay, measured from when the payment was logged.

Proposed configuration:
- Setting name: UnclaimedReminderDays
- Location: a shared configuration list
- Initial value: 3 days
- Recipients: to be confirmed

The location and initial delay are proposals requiring team confirmation.

The reminder should include the payment reference and direct recipients
to the Payment Log Register.

Reminder automation has not been implemented in this issue.

## Tests

All tests used fictional data in Établi.

| Test | Result |
|---|---|
| Save with Gift Amount missing | Passed — save refused |
| Save with all five required fields completed | Passed — payment saved |
| Initial status is Unclaimed | Passed |
| Test button generates a payment reference from the saved ID | Passed |
| Filter Status eq 'Unclaimed' excludes a Claimed payment | Passed |

## Deliverables

- Rebuild document: docs/rebuild/payment-log-registry.md
- Lab export committed under lab/solutions/
- Branch: feature/3-payment-log-data-and-rules

## Remaining work

- Confirm shared field names and the submission reference format
  with Dev 2's completed data dictionary.
