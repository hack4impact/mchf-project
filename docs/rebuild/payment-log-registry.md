# Payment Log — Data and Rules

Related issue: [#3](https://github.com/hack4impact/mchf-project/issues/3)
Related issue: [#10](https://github.com/hack4impact/mchf-project/issues/10)
Related issue: [#24](https://github.com/hack4impact/mchf-project/issues/24) 

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
| ConstituentSystemID | Constituent System ID | Single line of text | No | None |

Status choices are Unclaimed, Claimed and Coded.

The built-in Title column is retained.

Payment Type and Gift Type store text. The payment-entry app obtains
active options from Dev 2's MCHF Payment Types and MCHF Gift Types
lists, ordered by Sort Order. Both combo boxes allow one selection.

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
a reference. Reference generation runs through the reference test
button and the payment-entry app's Submit button.

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

Implemented in issue #24.

The flow `MCHF - Unclaimed reminder` runs daily at 9 a.m. Eastern time.
It selects Unclaimed payments whose Created date is at or before
the current time minus the configured delay.

### Settings

List: `MCHF - Settings`

| Internal name | Display name | Type | Required | Default |
|---|---|---|---|---|
| Title | Titre | Single line of text | Yes | None |
| SettingValue | Setting Value | Number, minimum 0, no decimals | Yes | None |

The list contains one setting:
- Title: `UnclaimedReminderDays`
- Setting Value: `3`

Create SettingValue without spaces first, then rename it to Setting Value.
The delay can be changed in this list without editing the flow.

### Flow steps

| Card | Action | Purpose |
|---|---|---|
| Recurrence | Scheduled trigger | Every day at 9:00, Eastern Standard Time |
| Get settings | Get items | Read the UnclaimedReminderDays setting |
| Delay days | Compose | Convert SettingValue to a number |
| Cutoff | Compose | Subtract the delay from the current time |
| Get overdue payments | Get items | Select Unclaimed payments at or before the cutoff, ordered by Created ascending |
| Initialize variable | Initialize variable | Create an empty String variable named lines |
| Any overdue | Condition | Check whether any payments were found |
| Each payment | Apply to each | In Si oui, append each payment's details to lines |
| Post reminder | Post message in a chat or channel | After the loop, post one message as Flow bot |

Si non is empty, so no message is posted when no payments are overdue.

### Reminder message

Lab destination: Teams channel `MCHF Lab / Fundraisers`.

The message includes the payment count, configured delay, and each
payment's reference, constituent name, amount and Created date.
If a reference is missing, the payment's ID is shown instead.

It directs fundraisers to open the Payment Log registry and claim
their payments. Production recipients remain to be confirmed with Marc.

### Reminder tests

| Test | Result |
|---|---|
| Delay 0 lists every Unclaimed payment and only those | Passed |
| Changing a payment to Claimed excludes it from the next reminder | Passed |
| No payments past the delay results in no new Teams message | Passed |
| Changing the setting changes the results without editing the flow | Passed |

The delay was restored to 3 after testing.

## Tests

All tests used fictional data in Établi.

| Test | Result |
|---|---|
| Save with Gift Amount missing | Passed — save refused |
| Save with all five required fields completed | Passed — payment saved |
| Initial status is Unclaimed | Passed |
| Test button generates a payment reference from the saved ID | Passed |
| Filter Status eq 'Unclaimed' excludes a Claimed payment | Passed |

## Payment entry and registry screens — issue #10

App name: MCHF - Payment Log
Environment: Équipe

Screens:
- scrPaymentEntry: look up a donor and log a payment.
- scrRegistry: view, search, filter and sort payments.

The app uses the MCHF - Payment Log list, Dev 2's MCHF Payment Types
and MCHF Gift Types lists, and the donor lookup contract from issue #2.

### Payment entry

Required:
- Valid donor lookup matching the entered Constituent ID.
- Payment Type.
- Gift Date.
- Gift Amount greater than zero, with at most two decimal places.

Optional:
- Gift Type.
- Assigned Solicitor.
- Notes.

Payment Type and Gift Type use single-selection combo boxes.
Their options come from the active entries in the configurable lists,
ordered by Sort Order.

Submit saves the donor's Constituent ID, Constituent Name and
Constituent System ID along with the payment details.

New payments receive status Unclaimed. Their PAY reference is generated
from the saved list ID using the two-Patch pattern described above.

After saving, the app displays the payment reference and resets the form.
Submission Reference remains blank until a submission is linked.

Cancel resets the entry fields, clears the donor and message,
and returns to scrRegistry without saving.

### Registry controls and filtering

Screen: scrRegistry
Gallery: galRegistry

Controls outside the gallery:

| Control | Property | Value |
|---|---|---|
| btnOpenEntry | Text | "+ Log a payment" |
| btnOpenEntry | OnSelect | Navigate(scrPaymentEntry) |
| txtRegistrySearch | Default | "" |
| txtRegistrySearch | HintText | "Search by ID or name" |
| ddRegistryStatus | Items | ["All", "Unclaimed", "Claimed", "Coded"] |
| ddRegistryStatus | Default | "All" |
| ddRegistrySort | Items | ["Gift Date: newest first", "Gift Date: oldest first"] |
| ddRegistrySort | Default | "Gift Date: newest first" |

Set galRegistry.Items to:

```powerfx
Sort(
    Filter(
        'MCHF - Payment Log',
        (
            IsBlank(Trim(txtRegistrySearch.Text)) ||
            StartsWith('Constituent ID', Trim(txtRegistrySearch.Text)) ||
            StartsWith('Constituent Name', Trim(txtRegistrySearch.Text))
        ) &&
        (
            ddRegistryStatus.Selected.Value = "All" ||
            Status.Value = ddRegistryStatus.Selected.Value
        )
    ),
    'Gift Date',
    If(
        ddRegistrySort.Selected.Value = "Gift Date: oldest first",
        SortOrder.Ascending,
        SortOrder.Descending
    )
)
```

Search matches the beginning of the Constituent ID or name.
Search and status filtering work together.

Sorting uses Gift Date, not the PAY reference.
Payments with equal gift dates have no additional tie-breaker.

### Registry columns and headings

Set galRegistry.Width to 1165 and TemplateSize to 50.

Add these controls inside the gallery.
For each control, set Y to 5 and Height to 35.

| Control | Text formula | X | Width |
|---|---|---:|---:|
| lblPaymentRef | ThisItem.Reference | 10 | 100 |
| lblRegistryLogged | Text(ThisItem.Created, "yyyy-mm-dd") | 120 | 110 |
| lblRegistryGiftDate | Text(ThisItem.'Gift Date', "yyyy-mm-dd") | 230 | 110 |
| lblRegistryDonorId | ThisItem.'Constituent ID' | 345 | 110 |
| lblRegistryDonorName | ThisItem.'Constituent Name' | 480 | 170 |
| lblRegistryAmount | Text(ThisItem.'Gift Amount', "0.00") | 645 | 80 |
| lblRegistryPaymentType | ThisItem.'Payment Type' | 730 | 150 |
| lblRegistryStatus | ThisItem.Status.Value | 885 | 110 |
| btnClaim | "Claim" | 1010 | 150 |
| lblClaimedBy | ThisItem.'Claimed By Name' | 1010 | 150 |

Logged displays the built-in Created date.
Gift Date displays the date entered on the payment form.

Create nine heading labels outside the gallery:
- Payment Ref.
- Logged
- Gift date
- Constituent ID
- Name
- Amount
- Payment Type
- Status
- Claimed by / action

For each heading:
- X: galRegistry.X + the corresponding column X.
- Y: galRegistry.Y - 40.
- Width: the corresponding column width.
- Height: 35.

### Status badges and Claim display

Set lblRegistryStatus.Fill to:

```powerfx
Switch(
    ThisItem.Status.Value,
    "Unclaimed", ColorValue("#FFF4CE"),
    "Claimed", ColorValue("#DEECF9"),
    "Coded", ColorValue("#DFF6DD"),
    ColorValue("#E5E7EB")
)
```

Set lblRegistryStatus.Color to:

```powerfx
ColorValue("#333333")
```

Set btnClaim.Visible to:

```powerfx
ThisItem.Status.Value = "Unclaimed"
```

Set btnClaim.OnSelect to:

```powerfx
Blank()
```

Set lblClaimedBy.Visible to:

```powerfx
ThisItem.Status.Value <> "Unclaimed"
```

Unclaimed payments display the Claim button.
Claimed and Coded payments display the saved claimant name instead.

The Claim button does not modify any records in issue #10.
The claiming action will be implemented in issue #14.

### Empty states

Add two labels outside galRegistry, in the same empty area.

lblRegistryEmpty.Text:

```powerfx
"No payments yet. Use '+ Log a payment' to create the first entry."
```

lblRegistryEmpty.Visible:

```powerfx
IsEmpty('MCHF - Payment Log')
```

lblRegistryNoResults.Text:

```powerfx
"No payments match your search or status filter."
```

lblRegistryNoResults.Visible:

```powerfx
!IsEmpty('MCHF - Payment Log') &&
IsEmpty(
    Filter(
        'MCHF - Payment Log',
        (
            IsBlank(Trim(txtRegistrySearch.Text)) ||
            StartsWith('Constituent ID', Trim(txtRegistrySearch.Text)) ||
            StartsWith('Constituent Name', Trim(txtRegistrySearch.Text))
        ) &&
        (
            ddRegistryStatus.Selected.Value = "All" ||
            Status.Value = ddRegistryStatus.Selected.Value
        )
    )
)
```

The first message appears when the Payment Log contains no payments.
The second appears when payments exist but none match the filters.

### Cancel button

Set btnCancel.OnSelect to:

```powerfx
Reset(txtConstituentId);
Reset(cbxPaymentType);
Reset(dpGiftDate);
Reset(txtGiftAmount);
Reset(cbxGiftType);
Reset(txtSolicitor);
Reset(txtNotes);
Set(varDonor, Blank());
Set(varSaveMessage, "");
Navigate(scrRegistry)
```

Set Fill to ColorValue("#E5E7EB").
Set Color to ColorValue("#374151").

Cancel returns fields to their defaults, clears the donor and message,
and returns to the registry without creating a payment.

### Entry layout

The entry screen uses a two-column layout.

Set lblConstituentName.Width to 300 to allow longer names.
Set txtNotes.Height to 70 and Mode to TextMode.MultiLine.

### Wireframe differences

- Assigned Solicitor uses a text input.
- The undefined donor-unknown checkbox is omitted.
- The sort dropdown offers Gift Date newest-first and oldest-first;
  the wireframe did not specify the dropdown options.
- Status badges use pale yellow for Unclaimed, blue for Claimed,
  and green for Coded instead of the wireframe's grey styling.

### Screen tests

Tests used fictional data in Établi.

| Test | Result |
|---|---|
| Look up a valid donor | Passed |
| Submit without a valid donor | Passed — save blocked |
| Submit without Payment Type | Passed — save blocked |
| Submit without Gift Date | Passed — save blocked |
| Submit with an empty amount | Passed — save blocked |
| Submit with amount abc | Passed — save blocked |
| Submit with amount 12.345 | Passed — save blocked |
| Submit with amount 0 | Passed — save blocked |
| Change Constituent ID without repeating lookup | Passed — save blocked |
| Save Cash payment for donor 1234567 with amount 25.50 | Passed |
| Saved payment has PAY reference and status Unclaimed | Passed |
| Saved donor ID, name and Constituent System ID are populated | Passed |
| Successful save displays confirmation and resets the form | Passed |
| Cancel clears the form and returns without saving | Passed |
| Registry displays payment details and Logged date | Passed |
| Search by donor name and Constituent ID | Passed |
| Clear search to restore rows | Passed |
| Filter by payment status | Passed |
| Search zzzz displays the no-results message | Passed |
| Sort Gift Date newest-first and oldest-first | Passed |
| Claimed payment displays claimant name without Claim button | Passed — PAY-00013 displayed Alex |
| Coded payment displays claimant name without Claim button | Passed — displayed Yanis |
| Completely empty Payment Log displays the first-entry message | Not run — existing test payments retained |

### Rebuilding the entry formulas

The lookup uses issue #2's donor lookup contract.
The Submit behaviour is described in the Payment entry section.

Retain the working lookup and Submit formulas in the fresh application
export. Their exact formulas are not reproduced in this document.

## Deliverables

- Rebuild document: docs/rebuild/payment-log-registry.md
- Lab export committed under lab/solutions/
- Branch: feature/3-payment-log-data-and-rules
- Reminder export: lab/solutions/mchf-reminder.solution.json
- Reminder branch: feature/24-unclaimed-reminder
- Screens branch: feature/10-payment-log-and-registry-screens

## Remaining work

- Export the latest Payment Log application and list under lab/solutions/.
- Commit and push the #10 changes, then open the PR with screenshots.
- Claim action wiring belongs to issue #14.
