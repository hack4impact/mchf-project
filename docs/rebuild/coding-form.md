# Coding Form: Simple Gift

Related issues: [#8](https://github.com/hack4impact/mchf-project/issues/8) (with follow-ups from PR #38), [#22](https://github.com/hack4impact/mchf-project/issues/22) (Package, Assigned Solicitor and Notes), [#23](https://github.com/hack4impact/mchf-project/issues/23) (Tax receipt section and receipt amount), [#25](https://github.com/hack4impact/mchf-project/issues/25) (New donor question), [#26](https://github.com/hack4impact/mchf-project/issues/26) (Same-donor check), [#53](https://github.com/hack4impact/mchf-project/issues/53) (Pledge payment)
App: **MCHF - Coding Form** (Équipe)
Lab export: `lab/solutions/mchf-coding-form.solution.json` (the app and manual test flow, with no list items)

A fundraiser enters one simple gift: the donor (or a new donor), the gift, its
coding (campaign, appeal, one fund or a split across several funds, optional package,
assigned solicitor and notes), and whether a tax receipt should be issued (with
optional benefit amount and calculated/editable receipt amount). The form saves it as
a submission with the status `Submitted`, gives it a reference `SUB-00001`, writes
its fund lines, and links it to the payment it codes when opened from the Payment Log or when the fundraiser claims a payment in the form. For a pledge payment, it also writes one linked line per pledge paid.

Lab only. Saving uses `Patch` on lab lists; the real Power Apps and SharePoint
version is rebuilt and tested in Sprint 4 (see section 7).

## 1. Contract

**Input:** an optional URL parameter `paymentId`, the `ID` of an item in
`MCHF - Payment Log`, for example `…/play/e/<environment>/a/<app>?paymentId=12`.
#14 builds that link. Without it, the form opens empty.

**Writes**, in this order. The form stops at the first write that fails and says
which one (section 4, `btnSubmit.OnSelect`):

| Step | List | Write |
|---|---|---|
| 1 | `MCHF Submissions` | New item, `Title` = `"NEW"`, every field below, `Processing status` = `Submitted` |
| 2 | `MCHF Submissions` | Same item, `Title` = `"SUB-" & Text(ID, "00000")` (data dictionary, #1) |
| 3 | `MCHF Fund Lines` | One line per fund: `Title` = fund code, `Submission` = lookup to the new item, `Submission Reference` = its SUB number, `Amount`. Without a split: one line, the chosen fund, the whole gift amount |
| 4 | `MCHF - Pledge Payments` | On Yes: one line per pledge with its ID, owner, balance, payment amount, and submission lookup and reference. |
| 5 | `MCHF - Payment Log` | When a payment was loaded through `paymentId` or claimed in the same-donor panel: `Submission Reference` = the SUB number |

Then a popup shows the SUB number.

**Submission fields:**

| Column | Value |
|---|---|
| `New Donor` | Yes/No value from `rdoNewDonor`, default No |
| `Constituent ID` | The ID found by the lookup; empty for a new donor (Gift Processing adds it before approving) |
| `Constituent Name` | The name returned by NXT, or the new donor's name as typed |
| `Constituent System ID` | The NXT system ID returned by the lookup, empty for a new donor |
| `Payment Type`, `Gift Type` | `Title` of the chosen `MCHF Payment Types` / `MCHF Gift Types` item |
| `Gift Date` | The date picker |
| `Gift Amount` | Typed amount |
| `Fees` | Typed fees, `0` when empty |
| `Net Amount` | `Gift Amount - Fees` (never reduced by `Benefit Amount`) |
| `Campaign`, `Appeal` | NXT `lookup_id` of the chosen item (e.g. `AC-2025-2026`, `EV-GOLF-2026`) |
| `Fund` | Primary fund: the chosen fund, or with a split the line with the largest amount (e.g. `HA-GENE0001`) |
| `Package` | Optional free-text package code (`Trim(txtPackage.Text)`), empty when omitted |
| `Assigned Solicitor` | Optional 7-digit solicitor Constituent ID (`Trim(txtSolicitor.Text)`), validated as an active fundraiser in RE NXT; prefilled from the payment when present, never from the signed-in user |
| `Notes` | Optional multiline gift notes (`txtNotes.Text`), prefilled from `varPaymentRecord.Notes` when present |
| `TaxReceipt` | Boolean (`tglTaxReceipt.Value`): `true` if a tax receipt is to be issued, `false` otherwise |
| `BenefitAmount` | Benefit amount (`Value(txtBenefitAmount.Text)`) when `TaxReceipt` is `true` and the input is non-empty; `Blank()` otherwise |
| `ReceiptAmount` | Receipt amount (`Value(txtReceiptAmount.Text)`), defaulting to `Gift Amount - Benefit Amount` and editable when `TaxReceipt` is `true`; `Blank()` when `false` |
| `Pledge Payment` | Boolean from `tglPledgePayment.Value`, default No |
| `Payment Reference` | The payment's `Reference` (e.g. `PAY-00012`), when there is one |

## 2. Screen

`Screen1`, tablet 1366 × 1080. Most labels are in French. The new-donor question and controls are in English; the donor and
solicitor lookup messages are the English ones of the #2 and #22 contracts.

- **Left column:** `txtConstituentId` + `btnSearchDonor` ("Lookup"); under them one
  line that shows the donor's name (`lblConstituentName`), a lookup message
  (`lblDonorMsg`) or the validation message (`lblErrorConstituentId`); the radio
  `rdoNewDonor` and the conditional controls in `conNewDonor` (section 4);
  `txtGiftAmount`, `txtFees`, `lblNetAmountText` / `lblNetAmount`; the tax receipt
  toggle `tglTaxReceipt` + `lblTaxReceiptQuestion` ("Reçu fiscal à émettre ?"),
  `lblBenefitAmount` + `txtBenefitAmount` ("Avantage ($)") and `lblReceiptAmount` +
  `txtReceiptAmount` ("Montant du reçu ($)") (visible only when
  `tglTaxReceipt.Value` is `true`), `lblErrTaxReceipt`.
- **Middle column:** `dpGiftDate`, `ddGiftType`, `ddPaymentType`, `ddCampaign`,
  `ddAppeal`, `ddFund` (hidden when the gift is split), and `txtPackage`
  ("Package (facultatif)").
- **Right column:** `txtSolicitor` ("Sollicitateur assigné (facultatif)") +
  `btnCheckSolicitor` ("Check"), `lblSolicitor` (active solicitor name in black or
  check error in red), `lblErrSolicitor` (Submit validation message), the toggle
  `tglSplitFunds`, then for a split: `ddLineFund` + `txtLineAmount`, `btnAddLine`
  ("Ajouter la ligne" / "Mettre à jour la ligne"), `btnCancelLine`, the gallery
  `galFundLines` (fund, amount, edit and delete icons), `lblFundTotal` and
  `lblErrFundLines`; at the bottom, `txtNotes` ("Notes (facultatif)", multiline).
- **Below the main columns:** the pledge payment toggle and conditional line editor (#53, section 4). Submit sits below that section.
- **Under every required field:** a red label `lblErr…`, visible only after a
  Submit attempt (`varTried`) while that field is invalid.
- **Success popup:** `rectOverlay`, `rectModalCard`, `lblSuccessTitle`,
  `lblSuccessRef`, `btnClosePopup` ("Fermer et réinitialiser"). They are the last
  controls of the screen, so they draw above everything else.

## 3. Rules

- **Required fields start empty.** No amount, date, type or code is chosen for the
  fundraiser: a field that nobody touched must block Submit. The only values that
  start filled come from the payment being coded. Fees start at `0.00` (optional).
- **Empty first choice.** A lab or Power Apps Dropdown whose `Default` matches no item
  selects the first item. So each list starts with a placeholder whose `code` is
  `""` ("Sélectionner…"), and validation checks `.Selected.code`, never
  `.Selected.Value` (the placeholder's text is not blank).
- **Options are never typed in the screen.** Payment and gift types come from
  `MCHF Payment Types` and `MCHF Gift Types` (`Active` only, by `Sort Order`), so the
  form offers exactly the values NXT mapping knows (`nxt-mapping.md`). Campaigns,
  appeals and funds come from `RENXT.ListCampaigns()`, `ListAppeals()` and
  `ListFunds()`; the form keeps the `lookup_id`.
- **Donor lookup (#2 contract).** Seven digits, then `SearchConstituents` with
  `search_field: "lookup_id"`, filtered on `lookup_id = typedId`. Stored in the
  global record `varDonor` (`{state, systemId, lookupId, name, message}`). Exactly
  one match: found. None: "No donor with this Constituent ID. Check the number."
  More than one: "Several donors match. Contact Gift Processing." An error or a
  timeout from NXT: "Donor lookup unavailable. Try again in a moment.", never "not
  found" (otherwise the fundraiser would switch to New donor and create a duplicate).
- **The saved ID is the one that was looked up.** Typing in the ID field clears
  `varDonor` (`OnChange`), and Submit also checks `varDonor.lookupId = Trim(txtConstituentId.Text)`.
- **Assigned Solicitor (#22 contract).** Optional 7-digit Constituent ID, never
  prefilled from the signed-in user. Clicking `btnCheckSolicitor` ("Check") searches
  RE NXT with `SearchConstituents(typedId, {search_field: "lookup_id", include_inactive: true})`:
  - Not 7 digits: `"Enter the solicitor's 7-digit Constituent ID."`
  - Not found (count <> 1): `"No solicitor with this ID."`
  - Found, but `fundraiser_status <> "Active"` or `inactive`: `"<Name> is not an active solicitor."`
  - Found and active (`fundraiser_status = "Active" && !inactive`): displays the
    solicitor's name in black and sets `varSolicitor.state = "found"`.
  - NXT error or timeout: `"Solicitor check unavailable. Try again in a moment."`
  - Typing in `txtSolicitor` clears `varSolicitor` (`OnChange`). If `txtSolicitor` is
    non-empty on Submit without `varSolicitor.state = "found"`, Submit is blocked and
    `lblErrSolicitor` displays `"Check the solicitor's ID, or leave it empty."`.
- **Package & Notes (#22 contract).** Both are optional. `txtPackage` is single-line
  free text; `txtNotes` is multiline free text, prefilled from `varPaymentRecord.Notes`
  when opened from the Payment Log, and saved to `MCHF Submissions.Notes`.
- **Tax receipt (#23 contract).** `tglTaxReceipt` ("Reçu fiscal à émettre ?")
  defaults to `false` (No). `txtBenefitAmount` and `txtReceiptAmount` are visible only
  when `tglTaxReceipt.Value` is `true`. `txtBenefitAmount` starts empty (treated as
  `0` by `Coalesce` when computing the receipt), and `txtReceiptAmount` defaults to
  `Max(0, Gift Amount - Benefit Amount)`. Changing `txtGiftAmount` or
  `txtBenefitAmount` resets and recalculates `txtReceiptAmount` (replacing any value
  typed manually), while `txtReceiptAmount` remains manually editable afterwards.
  Both amounts must stay between `0` and `Gift Amount` (up to two decimals); an
  out-of-range amount (such as a $120 benefit on a $100 gift) blocks Submit and
  shows `lblErrTaxReceipt`. Switching `tglTaxReceipt` back from Yes to No resets both
  inputs and saves `TaxReceipt = false` with `Blank()` for `BenefitAmount` and
  `ReceiptAmount`. `Net Amount` is never reduced by `Benefit Amount`.
- **New donor (#25, PRD acceptance #19).** “Is this a new donor?” defaults to No.
  Yes hides the ID field and Lookup, shows the individual and organization form
  buttons, and requires a typed name. Switching answers clears the lookup result
  and resets both donor inputs. On No, the lookup must match the current ID.
  Gift Processing adds a new donor’s ID before approval.
- **Fund split** (#8 priority 2; PRD: the fund amounts must equal the gift amount).
  "Répartir entre plusieurs fonds ?" replaces the Fund dropdown by lines. A line is
  added with a fund and an amount (more than 0, up to two decimals); the edit icon
  loads it back into the inputs ("Mettre à jour la ligne", or "Annuler"); the delete
  icon removes it. The same fund cannot be on two lines. Submit is blocked until
  there is at least one line and the lines add up to the gift amount (to the cent).
  The lines are kept in the collection `colFundLines` until the submission is saved.
- **Amounts.** Gift amount: digits with up to two decimals, more than 0. Fees: empty,
  or digits with up to two decimals and no more than the gift amount. The net amount
  shows only when both are valid. The gift amount is never changed.
- **Gift date.** Required; past dates are accepted (codings often arrive weeks after
  the gift). Without a payment it starts empty rather than on today, so a forgotten
  date is caught instead of saved silently.
- **Prefill from a payment.** Constituent ID (then looked up right away, as if the
  fundraiser had clicked Lookup, which fills the name), gift amount, gift date,
  payment type, gift type, assigned solicitor (prefilled as an ID in `txtSolicitor`
  for the fundraiser to verify with Check) and the payment's notes in `txtNotes`. A
  `paymentId` that matches no payment, or a payment that already has a
  `Submission Reference`, shows a warning; the second case is not blocked.
- **After a save.** "Fermer et réinitialiser" clears every field (including
  `txtPackage`, `txtSolicitor`, `txtNotes`, `tglTaxReceipt`, `txtBenefitAmount` and
  `txtReceiptAmount`), `varDonor`, `varSolicitor`, the fund lines and
  `varPaymentRecord`, so the next gift cannot be linked to the payment that was
  just coded.

## 4. Formulas

### `Screen1.OnVisible`

```powerfx
// 1. The payment being coded, when the form is opened from the Payment Log with ?paymentId=<item ID> (#14)
Set(varPaymentIdParam, Trim(Param("paymentId")));
Set(varPaymentRecord, If(IsBlank(varPaymentIdParam), Blank(), LookUp('MCHF - Payment Log', ID = IfError(Value(varPaymentIdParam), -1))));
If(!IsBlank(varPaymentIdParam) && IsBlank(varPaymentRecord),
    Notify("Payment " & varPaymentIdParam & " is not in the Payment Log. Fill in the form by hand.", NotificationType.Warning));
If(!IsBlank(varPaymentRecord.'Submission Reference'),
    Notify("This payment is already coded as " & varPaymentRecord.'Submission Reference' & ". Submitting again creates a second submission.", NotificationType.Warning));

// 2. Payment types and gift types from the #1 lists, with an empty first choice (code "")
ClearCollect(colPaymentTypes, {Value: "Sélectionner un mode de paiement...", code: ""});
Collect(colPaymentTypes, ForAll(Sort(Filter('MCHF Payment Types', Active), 'Sort Order'), {Value: Title, code: Title}));
ClearCollect(colGiftTypes, {Value: "Sélectionner un type de don...", code: ""});
Collect(colGiftTypes, ForAll(Sort(Filter('MCHF Gift Types', Active), 'Sort Order'), {Value: Title, code: Title}));

// 3. Campaigns, appeals and funds from RE NXT, with an empty first choice (code "")
IfError(
    ClearCollect(colCampaigns, {Value: "Sélectionner une campagne...", code: ""});
    Collect(colCampaigns, ForAll(RENXT.ListCampaigns().value, {Value: ThisRecord.lookup_id & " - " & ThisRecord.description, code: ThisRecord.lookup_id}));
    ClearCollect(colAppeals, {Value: "Sélectionner un appel...", code: ""});
    Collect(colAppeals, ForAll(RENXT.ListAppeals().value, {Value: ThisRecord.lookup_id & " - " & ThisRecord.description, code: ThisRecord.lookup_id}));
    ClearCollect(colFunds, {Value: "Sélectionner un fonds...", code: ""});
    Collect(colFunds, ForAll(RENXT.ListFunds().value, {Value: ThisRecord.lookup_id & " - " & ThisRecord.description, code: ThisRecord.lookup_id})),
    Notify("NXT lists are unavailable. Reopen the form to retry.", NotificationType.Error)
);

// 4. Screen state; colFundLines holds the lines of a gift split across several funds
UpdateContext({varTried: false, varSubmissionRef: "", varShowSuccessPopup: false, varEditLineId: 0, varLineError: ""});
Set(varDonor, Blank());
Set(varSolicitor, Blank());
ClearCollect(colFundLines, {lineId: 0, fund: "", fundLabel: "", amount: 0});
Clear(colFundLines);

// 5. A prefilled donor is looked up right away (same lookup as the Lookup button, #2 contract)
If(!IsBlank(varPaymentRecord),
    With({typedId: Trim(varPaymentRecord.'Constituent ID')},
        If(IsMatch(typedId, "\d{7}"),
            Set(varDonor, IfError(
                With({r: Filter(RENXT.SearchConstituents(typedId, {search_field: "lookup_id"}).value, lookup_id = typedId)},
                    Switch(CountRows(r),
                        0, {state: "notfound", systemId: "", lookupId: "", name: "", message: "No donor with this Constituent ID. Check the number."},
                        1, {state: "found", systemId: Text(First(r).id), lookupId: First(r).lookup_id, name: First(r).name, message: ""},
                        {state: "error", systemId: "", lookupId: "", name: "", message: "Several donors match. Contact Gift Processing."})),
                {state: "error", systemId: "", lookupId: "", name: "", message: "Donor lookup unavailable. Try again in a moment."})))));
Set(varNewIndividualUrl, "https://example.org/new-individual");
Set(varNewOrganizationUrl, "https://example.org/new-organization")
```

### New donor (#25)

| Control | Property | Formula |
|---|---|---|
| `lblNewDonator` | `Text` | `"Is this a new donor?"` |
| `rdoNewDonor` | `Items` | `["No", "Yes"]` |
| `rdoNewDonor` | `Default` | `"No"` |
| `rdoNewDonor` | `OnChange` | `Set(varDonor, Blank()); Reset(txtConstituentId); Reset(txtNewDonorName)` |
| `conNewDonor` | `Visible` | `rdoNewDonor.Selected.Value = "Yes"` |
| `btnNewIndividual` | `OnSelect` | `Launch(varNewIndividualUrl)` |
| `btnNewOrganization` | `OnSelect` | `Launch(varNewOrganizationUrl)` |
| `txtNewDonorName` | `HintText` | `"Donor name, as on the cheque"` |
| `lblDonatorID`, `txtConstituentId`, `btnSearchDonor` | `Visible` | `rdoNewDonor.Selected.Value = "No"` |

Place both buttons, the name input, and its error label inside `conNewDonor`.
Label the buttons “New individual donor form” and “New organization donor form”.

### `btnSearchDonor.OnSelect`

```powerfx
With({typedId: Trim(txtConstituentId.Text)},
    If(!IsMatch(typedId, "\d{7}"),
        Set(varDonor, {state: "invalid", systemId: "", lookupId: "", name: "", message: "Enter the 7-digit Constituent ID."}),
        Set(varDonor, IfError(
            With({r: Filter(RENXT.SearchConstituents(typedId, {search_field: "lookup_id"}).value, lookup_id = typedId)},
                Switch(CountRows(r),
                    0, {state: "notfound", systemId: "", lookupId: "", name: "", message: "No donor with this Constituent ID. Check the number."},
                    1, {state: "found", systemId: Text(First(r).id), lookupId: First(r).lookup_id, name: First(r).name, message: ""},
                    {state: "error", systemId: "", lookupId: "", name: "", message: "Several donors match. Contact Gift Processing."})),
            {state: "error", systemId: "", lookupId: "", name: "", message: "Donor lookup unavailable. Try again in a moment."}))));
Refresh('MCHF - Payment Log');
ClearCollect(
    colSameDonor,
    ForAll(
        Filter(
            'MCHF - Payment Log',
            'Constituent ID' = varDonor.lookupId && Status.Value = "Unclaimed"
        ),
        {
            ID: ThisRecord.ID,
            Reference: ThisRecord.Reference,
            'Gift Amount': ThisRecord.'Gift Amount',
            'Gift Date': ThisRecord.'Gift Date',
            'Payment Type': ThisRecord.'Payment Type'
        }
    )
);
Set(varSameDonorDone, false)
```

`txtConstituentId.OnChange`:

```powerfx
Set(varDonor, Blank())
```

### Same-donor check (#26)

After a successful donor lookup on a blank form, show the donor’s Unclaimed payments. The lookup above refreshes the Payment Log and copies the payment ID and four display fields into `colSameDonor`. Each new lookup resets the dismissal flag.

| Control | Property | Formula |
|---|---|---|
| `conSameDonor` | `Visible` | `varDonor.state = "found" && IsBlank(varPaymentRecord) && !varSameDonorDone && !IsEmpty(colSameDonor)` |
| `lblSameDonor` | `Text` | `"This donor has payments waiting to be claimed. Is one of them your gift?"` |
| `galSameDonor` | `Items` | `colSameDonor` |
| `lblSameReference` | `Text` | `ThisItem.Reference` |
| `lblSameAmount` | `Text` | `Text(ThisItem.'Gift Amount', "0.00")` |
| `lblSameDate` | `Text` | `ThisItem.'Gift Date'` |
| `lblSamePaymentType` | `Text` | `ThisItem.'Payment Type'` |
| `btnClaimThis` | `Text` | `"Claim this payment"` |
| `btnNoneMine` | `Text` | `"None of these is my gift"` |
| `btnNoneMine` | `OnSelect` | `Set(varSameDonorDone, true)` |

Place the four payment labels and `btnClaimThis` inside the gallery row. Place the prompt above the gallery and `btnNoneMine` below it inside the container. Dismissing the panel leaves the fundraiser on the blank form.

`btnClaimThis.OnSelect` refreshes the list and looks up the selected payment by ID. A successful claim stores the updated record in `varPaymentRecord`, which supplies the existing field defaults and the payment reference on Submit.

```powerfx
With(
    {paymentIdToClaim: ThisItem.ID},
    IfError(
        Refresh('MCHF - Payment Log');
        With(
            {current: LookUp('MCHF - Payment Log', ID = paymentIdToClaim)},
            If(
                IsBlank(current),
                Notify("This payment no longer exists.", NotificationType.Error);
                Remove(colSameDonor, ThisItem),
                current.Status.Value <> "Unclaimed",
                Notify(
                    If(
                        current.Status.Value = "Claimed" && !IsBlank(current.'Claimed By Name'),
                        "Already claimed by " & current.'Claimed By Name' & ".",
                        "This payment is no longer Unclaimed."
                    ) & " Choose another payment or continue without it.",
                    NotificationType.Error
                );
                Remove(colSameDonor, ThisItem),
                With(
                    {
                        claimed: Patch(
                            'MCHF - Payment Log',
                            current,
                            {
                                Status: {Value: "Claimed"},
                                'Claimed By Name': User().FullName,
                                'Claimed By Email': User().Email,
                                'Claimed At': Now()
                            }
                        )
                    },
                    Set(varPaymentRecord, claimed);
                    Set(varSameDonorDone, true);
                    Notify(
                        "Payment " & claimed.Reference & " claimed. The form now shows it.",
                        NotificationType.Success
                    )
                )
            )
        ),
        Notify(
            "The claim could not finish. Check the Payment Log before retrying.",
            NotificationType.Error
        )
    )
)
```

### `btnCheckSolicitor.OnSelect`

```powerfx
With({typedId: Trim(txtSolicitor.Text)},
  If(IsBlank(typedId),
    Set(varSolicitor, Blank()),
    !IsMatch(typedId, "\d{7}"),
    Set(varSolicitor, {state: "invalid", name: "", message: "Enter the solicitor's 7-digit Constituent ID."}),
    Set(varSolicitor, IfError(
      With({r: Filter(RENXT.SearchConstituents(typedId, {search_field: "lookup_id", include_inactive: true}).value, lookup_id = typedId)},
        If(CountRows(r) <> 1,
          {state: "notfound", name: "", message: "No solicitor with this ID."},
          First(r).fundraiser_status = "Active" && !First(r).inactive,
          {state: "found", name: First(r).name, message: ""},
          {state: "refused", name: First(r).name, message: First(r).name & " is not an active solicitor."}
        )
      ),
      {state: "error", name: "", message: "Solicitor check unavailable. Try again in a moment."}
    ))
  )
)
```

`txtSolicitor.OnChange`:

```powerfx
Set(varSolicitor, Blank())
```

### Tax receipt (#23)

`txtGiftAmount.OnChange`:

```powerfx
If(tglTaxReceipt.Value, Reset(txtReceiptAmount))
```

`tglTaxReceipt.OnChange`:

```powerfx
If(!Self.Value, Reset(txtBenefitAmount); Reset(txtReceiptAmount))
```

`txtBenefitAmount.OnChange`:

```powerfx
Reset(txtReceiptAmount)
```

`txtReceiptAmount.Default`:

```powerfx
If(
    IsMatch(Trim(txtGiftAmount.Text), "^\d+(\.\d{1,2})?$") && Value(txtGiftAmount.Text) > 0 &&
    (IsBlank(Trim(txtBenefitAmount.Text)) || IsMatch(Trim(txtBenefitAmount.Text), "^\d+(\.\d{1,2})?$")),
    Text(Max(0, Value(txtGiftAmount.Text) - Coalesce(Value(txtBenefitAmount.Text), 0)), "0.00"),
    ""
)
```

### Pledge payment (#53)

Add `MCHF - Pledge Payments` as an app data source. Re-add `MCHF Submissions`
in Données if Studio still uses a schema without the `Pledge Payment`
Yes/No column. The pledge payment list’s `Submission` column is a lookup
to `MCHF Submissions`.

Place the toggle below the existing Yes/No sections. The pledge group is
visible when `tglPledgePayment.Value` is true. Put Submit below the group
when it is open.

| Control | Settings |
|---|---|
| `tglPledgePayment` / `lblPledgePayment` | Default `false`. Label: Is this a pledge payment? |
| `txtPledgeConstituent` | Default `Coalesce(varDonor.lookupId, "")`. Accepts a different seven-digit constituent ID for a third-party pledge. |
| `btnFindPledges` | Find open pledges. Uses a separate constituent lookup and leaves the gift's donor unchanged. |
| `ddPledge` | Items `colOpenPledges`. Default `"Choose an open pledge"`. Its `Value` field displays the pledge ID, date, original amount and remaining balance. |
| `txtPledgeAmount` | Amount paid on the pledge. In Edit mode it defaults to the amount on the selected line. |
| `btnAddPledgeLine` | Add pledge / Update pledge, depending on `varPledgeEditLineId`. |
| `galPledgeLines` | Items `colPledgeLines`. Show `ThisItem.pledgeLabel` and `Text(ThisItem.amount, "$#,##0.00")`, with edit and delete icons. |

While editing an amount, set the constituent input, lookup button and
dropdown to `DisplayMode.Disabled` using
`If(varPledgeEditLineId = 0, DisplayMode.Edit, DisplayMode.Disabled)`.
To change the pledge on a line, delete the line and add the replacement.

Initialize the pledge collections and state in `Screen1.OnVisible`
immediately after `Clear(colFundLines);`:

```powerfx
ClearCollect(colOpenPledges, {Value: "", pledgeId: "", balance: 0});
Clear(colOpenPledges);
ClearCollect(colPledgeLines, {lineId: 0, pledgeId: "", pledgeLabel: "", constituentId: "", constituentName: "", balance: 0, amount: 0});
Clear(colPledgeLines);
UpdateContext({
    varPledgeEditLineId: 0,
    varPledgeLineError: "",
    varPledgeOwnerId: "",
    varPledgeOwnerName: ""
});
```

`txtPledgeConstituent.OnChange`

```powerfx
Clear(colOpenPledges);
Reset(ddPledge);
UpdateContext({varPledgeOwnerId: "", varPledgeOwnerName: "", varPledgeLineError: ""})
```

`tglPledgePayment.OnChange`

```powerfx
If(
    !Self.Value,
    Clear(colOpenPledges);
    Clear(colPledgeLines);
    UpdateContext({
        varPledgeEditLineId: 0,
        varPledgeLineError: "",
        varPledgeOwnerId: "",
        varPledgeOwnerName: ""
    });
    Reset(txtPledgeConstituent);
    Reset(ddPledge);
    Reset(txtPledgeAmount)
)
```

`btnFindPledges.OnSelect`

```powerfx
Clear(colOpenPledges); Reset(ddPledge); UpdateContext({varPledgeOwnerId: "", varPledgeOwnerName: "", varPledgeLineError: ""});  With(
    {typedId: Trim(txtPledgeConstituent.Text)},
    If(
        !IsMatch(typedId, "\d{7}"),
        UpdateContext({varPledgeLineError: "Enter the 7-digit Constituent ID."}),
        IfError(
            With(
                {
                    matches: Filter(
                        RENXT.SearchConstituents(typedId, {search_field: "lookup_id"}).value,
                        lookup_id = typedId
                    )
                },
                Switch(
                    CountRows(matches),
                    0, UpdateContext({
                        varPledgeLineError: "No donor with this Constituent ID. Check the number."
                    }),
                    1,
                    With(
                        {owner: First(matches)},
                        With(
                            {
                                openPledges: Filter(
                                    RENXT.ListGifts({
                                        constituent_id: Text(owner.id),
                                        gift_type: "Pledge"
                                    }).value,
                                    balance.value > 0
                                )
                            },
                            If(
                                IsEmpty(openPledges),
                                UpdateContext({
                                    varPledgeLineError: "No open pledge for this constituent."
                                }),
                                UpdateContext({
                                    varPledgeOwnerId: owner.lookup_id,
                                    varPledgeOwnerName: owner.name
                                });
                                ClearCollect(
                                    colOpenPledges,
                                    {Value: "Choose an open pledge", pledgeId: "", balance: 0}
                                );
                                ForAll(

                                   openPledges As pledge,

                                   Collect(

                                       colOpenPledges,

                                       {

                                           Value: "#" & Text(pledge.id) & " · " & Left(Text(pledge.date), 10) &

                                               " · " & Text(pledge.amount.value, "0.00") &

                                               " $ · balance " & Text(pledge.balance.value, "0.00") & " $",

                                           pledgeId: Text(pledge.id),

                                           balance: pledge.balance.value

                                       }

                                   );

                                   true

                               );
                                Reset(ddPledge)
                            )
                        )
                    ),
                    UpdateContext({
                        varPledgeLineError: "Several donors match. Contact Gift Processing."
                    })
                )
            ),
            UpdateContext({
                varPledgeLineError: "Donor or pledge lookup unavailable. Try again in a moment."
            })
        )
    ) )
```

`txtPledgeAmount.Default`:

```powerfx
If(varPledgeEditLineId = 0, "", Text(LookUp(colPledgeLines, lineId = varPledgeEditLineId).amount, "0.00"))
```

`btnAddPledgeLine.OnSelect`

```powerfx
With(
    {
        enteredAmount: Trim(txtPledgeAmount.Text),
        paid: IfError(Value(Trim(txtPledgeAmount.Text)), 0),
        existingLine: LookUp(colPledgeLines, lineId = varPledgeEditLineId)
    },
    With(
        {
            pledgeId: If(varPledgeEditLineId = 0, ddPledge.Selected.pledgeId, existingLine.pledgeId),
            pledgeBalance: If(varPledgeEditLineId = 0, ddPledge.Selected.balance, existingLine.balance)
        },
        If(
            IsBlank(pledgeId) ||
                (varPledgeEditLineId = 0 && varPledgeOwnerId <> Trim(txtPledgeConstituent.Text)),
            UpdateContext({varPledgeLineError: "Choose an open pledge."}),
             !IsMatch(enteredAmount, "^\d+(\.\d{1,2})?$") || paid <= 0,
            UpdateContext({
                varPledgeLineError: "Enter the amount paid on this pledge, for example 100.00."
            }),
             paid > pledgeBalance,
            UpdateContext({
                varPledgeLineError: "The amount is above this pledge's balance (" &
                    Text(pledgeBalance, "0.00") & " $)."
            }),
             varPledgeEditLineId = 0 &&
                !IsBlank(LookUp(colPledgeLines As line, line.pledgeId = ddPledge.Selected.pledgeId)),
            UpdateContext({
                varPledgeLineError: "This pledge is already in the list: edit its line."
            }),
             If(
                varPledgeEditLineId = 0,
                Collect(
                    colPledgeLines,
                    {
                        lineId: Coalesce(Max(colPledgeLines, lineId), 0) + 1,
                        pledgeId: pledgeId,
                        pledgeLabel: ddPledge.Selected.Value,
                        constituentId: varPledgeOwnerId,
                        constituentName: varPledgeOwnerName,
                        balance: pledgeBalance,
                        amount: paid
                    }
                ),
                Patch(colPledgeLines, existingLine, {amount: paid})
            );
            UpdateContext({varPledgeEditLineId: 0, varPledgeLineError: ""});
            Reset(ddPledge);
            Reset(txtPledgeAmount)
        )
    ) )
```

`icoEditPledgeLine.OnSelect`

```powerfx
UpdateContext({varPledgeEditLineId: ThisItem.lineId, varPledgeLineError: ""}); Reset(txtPledgeAmount)
```

`icoDeletePledgeLine.OnSelect`

```powerfx
If(
    varPledgeEditLineId = ThisItem.lineId,
    UpdateContext({varPledgeEditLineId: 0});
    Reset(txtPledgeAmount) ); Remove(colPledgeLines, ThisItem); UpdateContext({varPledgeLineError: ""})
```

Add these red validation labels:

| Label | Text | Visible |
|---|---|---|
| `lblPledgeLineError` | `varPledgeLineError` | `tglPledgePayment.Value && !IsBlank(varPledgeLineError)` |
| `lblErrPledgeGiftType` | `"A pledge payment needs the gift type Pledge Payment."` | `tglPledgePayment.Value && ddGiftType.Selected.code <> "Pledge Payment"` |
| `lblErrPledgeTotal` | `"The pledge lines must total the gift amount."` | `varTried && tglPledgePayment.Value && ddGiftType.Selected.code = "Pledge Payment" && !(CountRows(colPledgeLines) > 0 && Abs(Sum(colPledgeLines, amount) - IfError(Value(txtGiftAmount.Text), 0)) < 0.005)` |

The Submit and reset formulas are below. The form stores the pledge links
in the lab lists. This feature leaves the simulated NXT balances as
recorded, and Gift Processing applies the payment in NXT.

**Test data flow.** In Établi, run the manual flow
`MCHF - Create test pledges` using the flow owner's RE NXT connection.
It selects the first fund by its system ID, creates $300 and $200 pledges
for system ID `11001` (lookup ID `1234567`), a $500 pledge for system ID
`11003` (lookup ID `9999002`), and a $100 pledge for `11001`.
A final cash pledge payment links to the $100 pledge and fully pays it.
Each run creates a fresh set of test data in the shared simulator.
Enable the manual flow in Équipe after merging the change request.

### `btnSubmit.OnSelect`

```powerfx
UpdateContext({varTried: true});
If(
    ((rdoNewDonor.Selected.Value = "Yes") || (varDonor.state = "found" && varDonor.lookupId = Trim(txtConstituentId.Text))) &&
    ((rdoNewDonor.Selected.Value = "No") || !IsBlank(Trim(txtNewDonorName.Text))) &&
    !IsBlank(dpGiftDate.SelectedDate) &&
    !IsBlank(ddGiftType.Selected.code) &&
    !IsBlank(ddPaymentType.Selected.code) &&
    !IsBlank(ddCampaign.Selected.code) &&
    !IsBlank(ddAppeal.Selected.code) &&
    (tglSplitFunds.Value || !IsBlank(ddFund.Selected.code)) &&
    (IsMatch(Trim(txtGiftAmount.Text), "^\d+(\.\d{1,2})?$") && Value(txtGiftAmount.Text) > 0) &&
    (IsBlank(Trim(txtFees.Text)) || (IsMatch(Trim(txtFees.Text), "^\d+(\.\d{1,2})?$") && (!(IsMatch(Trim(txtGiftAmount.Text), "^\d+(\.\d{1,2})?$") && Value(txtGiftAmount.Text) > 0) || Value(txtFees.Text) <= Value(txtGiftAmount.Text)))) &&
    (!tglSplitFunds.Value || (CountRows(colFundLines) > 0 && (IsMatch(Trim(txtGiftAmount.Text), "^\d+(\.\d{1,2})?$") && Value(txtGiftAmount.Text) > 0) && Abs(Sum(colFundLines, amount) - Value(txtGiftAmount.Text)) < 0.005)) &&
    (IsBlank(Trim(txtSolicitor.Text)) || varSolicitor.state = "found") &&
    (!tglPledgePayment.Value || (ddGiftType.Selected.code = "Pledge Payment" && CountRows(colPledgeLines) > 0 && Abs(Sum(colPledgeLines, amount) - IfError(Value(txtGiftAmount.Text), 0)) < 0.005)) &&
    (!tglTaxReceipt.Value || (
        (IsBlank(Trim(txtBenefitAmount.Text)) || (IsMatch(Trim(txtBenefitAmount.Text), "^\d+(\.\d{1,2})?$") && Value(txtBenefitAmount.Text) >= 0 && (!(IsMatch(Trim(txtGiftAmount.Text), "^\d+(\.\d{1,2})?$") && Value(txtGiftAmount.Text) > 0) || Value(txtBenefitAmount.Text) <= Value(txtGiftAmount.Text)))) &&
        (IsMatch(Trim(txtReceiptAmount.Text), "^\d+(\.\d{1,2})?$") && Value(txtReceiptAmount.Text) >= 0 && (!(IsMatch(Trim(txtGiftAmount.Text), "^\d+(\.\d{1,2})?$") && Value(txtGiftAmount.Text) > 0) || Value(txtReceiptAmount.Text) <= Value(txtGiftAmount.Text)))
    )),
    // 1. Save with a temporary Title, 2. set the reference SUB-00001 from the new ID (#1 data dictionary),
    // 3. save fund lines, 4. save pledge lines if applicable, 5. link the payment.
    // Stops at the first failure. The submission's Fund is the primary fund: the largest line.
    With({lines: If(tglSplitFunds.Value, colFundLines,
            Table({lineId: 1, fund: ddFund.Selected.code, fundLabel: ddFund.Selected.Value, amount: Value(txtGiftAmount.Text)}))},
    With({created: Patch('MCHF Submissions', Defaults('MCHF Submissions'), {
            Title: "NEW",
            'New Donor': (rdoNewDonor.Selected.Value = "Yes"),
            'Constituent ID': If((rdoNewDonor.Selected.Value = "Yes"), Blank(), varDonor.lookupId),
            'Constituent Name': If((rdoNewDonor.Selected.Value = "Yes"), Trim(txtNewDonorName.Text), varDonor.name),
            'Constituent System ID': If((rdoNewDonor.Selected.Value = "Yes"), Blank(), varDonor.systemId),
            'Payment Type': ddPaymentType.Selected.code,
            'Gift Date': dpGiftDate.SelectedDate,
            'Gift Type': ddGiftType.Selected.code,
            'Pledge Payment': tglPledgePayment.Value,
            'Gift Amount': Value(txtGiftAmount.Text),
            Fees: Coalesce(Value(txtFees.Text), 0),
            'Net Amount': Value(txtGiftAmount.Text) - Coalesce(Value(txtFees.Text), 0),
            Campaign: ddCampaign.Selected.code,
            Appeal: ddAppeal.Selected.code,
            Fund: First(SortByColumns(lines, "amount", SortOrder.Descending)).fund,
            Package: Trim(txtPackage.Text),
            'Assigned Solicitor': Trim(txtSolicitor.Text),
            Notes: txtNotes.Text,
            'Tax Receipt': tglTaxReceipt.Value,
            'Receipt Amount': If(tglTaxReceipt.Value, Value(txtReceiptAmount.Text), Blank()),
            'Benefit Amount': If(tglTaxReceipt.Value && !IsBlank(Trim(txtBenefitAmount.Text)), Value(txtBenefitAmount.Text), Blank()),
            'Payment Reference': varPaymentRecord.Reference,
            'Processing status': {Value: "Submitted"}
        })},
        With({ref: "SUB-" & Text(created.ID, "00000")},
            IfError(
                created,
                    Notify("The submission was not saved: " & FirstError.Message, NotificationType.Error),
                Patch('MCHF Submissions', created, {Title: ref}),
                    Notify("Item " & created.ID & " was saved, but its reference could not be set: " & FirstError.Message, NotificationType.Error),
                ForAll(lines As line, Patch('MCHF Fund Lines', Defaults('MCHF Fund Lines'), {Title: line.fund, Submission: {Id: created.ID, Value: ref}, 'Submission Reference': ref, Amount: line.amount})),
                    Notify(ref & " was saved, but its fund lines were not: " & FirstError.Message, NotificationType.Error),
                If(
                    tglPledgePayment.Value,
                    ForAll(
                        colPledgeLines As pledgeLine,
                        Patch(
                            'MCHF - Pledge Payments',
                            Defaults('MCHF - Pledge Payments'),
                            {
                                Title: pledgeLine.pledgeId,
                                'Pledge ID': pledgeLine.pledgeId,
                                'Pledge Constituent ID': pledgeLine.constituentId,
                                'Pledge Constituent Name': pledgeLine.constituentName,
                                'Pledge Balance': pledgeLine.balance,
                                Amount: pledgeLine.amount,
                                Submission: {Id: created.ID, Value: ref},
                                'Submission Reference': ref
                            }
                        )
                    )
                ),
                    Notify(ref & " was saved, but its pledge lines were not: " & FirstError.Message, NotificationType.Error),
                If(!IsBlank(varPaymentRecord), Patch('MCHF - Payment Log', varPaymentRecord, {'Submission Reference': ref})),
                    Notify(ref & " was saved, but the payment could not be linked to it: " & FirstError.Message, NotificationType.Error),
                UpdateContext({varSubmissionRef: ref, varShowSuccessPopup: true, varTried: false});
                Notify("Soumission réussie. Référence : " & ref, NotificationType.Success))))),
    Notify("Some fields need your attention.", NotificationType.Error)
)
```

`IfError(Value1, Fallback1, Value2, Fallback2, …, DefaultResult)` evaluates the
values in order and stops at the first error, returning its fallback; the default
result runs only when every step succeeded. `created` is computed once by `With`.

### Fund split

`btnAddLine.OnSelect`:

```powerfx
With({amt: Trim(txtLineAmount.Text), code: ddLineFund.Selected.code},
    If(
        IsBlank(code),
            UpdateContext({varLineError: "Choisissez le fonds de la ligne."}),
        !IsMatch(amt, "^\d+(\.\d{1,2})?$") || Value(amt) <= 0,
            UpdateContext({varLineError: "Montant de la ligne : un nombre supérieur à 0, par exemple 60.00."}),
        CountRows(Filter(colFundLines, fund = code && lineId <> varEditLineId)) > 0,
            UpdateContext({varLineError: "Ce fonds est déjà dans la répartition : modifiez sa ligne."}),
        varEditLineId = 0,
            Collect(colFundLines, {lineId: Coalesce(Max(colFundLines, lineId), 0) + 1, fund: code, fundLabel: ddLineFund.Selected.Value, amount: Value(amt)});
            UpdateContext({varLineError: ""});
            Reset(ddLineFund); Reset(txtLineAmount),
        Patch(colFundLines, LookUp(colFundLines, lineId = varEditLineId), {fund: code, fundLabel: ddLineFund.Selected.Value, amount: Value(amt)});
            UpdateContext({varEditLineId: 0, varLineError: ""});
            Reset(ddLineFund); Reset(txtLineAmount)))
```

Gallery `galFundLines` (`Items` = `colFundLines`): `icoEditLine.OnSelect` =
`UpdateContext({varEditLineId: ThisItem.lineId, varLineError: ""})`, `icoDeleteLine.OnSelect`:

```powerfx
If(varEditLineId = ThisItem.lineId, UpdateContext({varEditLineId: 0}); Reset(ddLineFund); Reset(txtLineAmount));
Remove(colFundLines, ThisItem)
```

`btnCancelLine.OnSelect` = `UpdateContext({varEditLineId: 0, varLineError: ""}); Reset(ddLineFund); Reset(txtLineAmount)`

### `btnClosePopup.OnSelect`

```powerfx
UpdateContext({varShowSuccessPopup: false, varSubmissionRef: "", varTried: false, varEditLineId: 0, varLineError: ""});
// The next gift is entered by hand: forget the payment, donor and solicitor that were just coded
Set(varPaymentRecord, Blank());
Set(varDonor, Blank());
Set(varSolicitor, Blank());
Clear(colFundLines);
Reset(txtConstituentId); Reset(rdoNewDonor); Reset(txtNewDonorName); Reset(txtGiftAmount); Reset(txtFees); Reset(dpGiftDate);
Reset(ddGiftType); Reset(ddPaymentType); Reset(ddCampaign); Reset(ddAppeal); Reset(ddFund);
Reset(txtPackage); Reset(txtSolicitor); Reset(txtNotes);
Reset(tglTaxReceipt); Reset(txtBenefitAmount); Reset(txtReceiptAmount);
Reset(tglSplitFunds); Reset(ddLineFund); Reset(txtLineAmount);
Clear(colPledgeLines);
Clear(colOpenPledges);
UpdateContext({
    varPledgeEditLineId: 0,
    varPledgeLineError: "",
    varPledgeOwnerId: "",
    varPledgeOwnerName: ""
});
Reset(tglPledgePayment);
Reset(txtPledgeConstituent);
Reset(ddPledge);
Reset(txtPledgeAmount)
```

### Defaults

| Control | Property | Formula |
|---|---|---|
| `txtConstituentId` | `Default` | `If(IsBlank(varPaymentRecord), "", varPaymentRecord.'Constituent ID')` |
| `txtConstituentId` | `DisplayMode` | `If((rdoNewDonor.Selected.Value = "Yes"), DisplayMode.Disabled, DisplayMode.Edit)` |
| `txtGiftAmount` | `Default` | `If(IsBlank(varPaymentRecord), "", Text(varPaymentRecord.'Gift Amount', "0.00"))` |
| `txtFees` | `Default` | `"0.00"` |
| `tglTaxReceipt` | `Default` | `false` |
| `lblBenefitAmount` | `Visible` | `tglTaxReceipt.Value` |
| `txtBenefitAmount` | `Default` | `""` |
| `txtBenefitAmount` | `Visible` | `tglTaxReceipt.Value` |
| `lblReceiptAmount` | `Visible` | `tglTaxReceipt.Value` |
| `txtReceiptAmount` | `Default` | `If(IsMatch(Trim(txtGiftAmount.Text), "^\d+(\.\d{1,2})?$") && Value(txtGiftAmount.Text) > 0 && (IsBlank(Trim(txtBenefitAmount.Text)) \|\| IsMatch(Trim(txtBenefitAmount.Text), "^\d+(\.\d{1,2})?$")), Text(Max(0, Value(txtGiftAmount.Text) - Coalesce(Value(txtBenefitAmount.Text), 0)), "0.00"), "")` |
| `txtReceiptAmount` | `Visible` | `tglTaxReceipt.Value` |
| `dpGiftDate` | `DefaultDate` | `If(IsBlank(varPaymentRecord), Blank(), varPaymentRecord.'Gift Date')` |
| `ddGiftType` | `Default` | `Coalesce(varPaymentRecord.'Gift Type', "")` |
| `ddPaymentType` | `Default` | `Coalesce(varPaymentRecord.'Payment Type', "")` |
| `ddCampaign` | `Default` | `""` |
| `ddAppeal` | `Default` | `""` |
| `ddFund` | `Default` | `""` |
| `ddFund` | `Visible` | `!tglSplitFunds.Value` |
| `txtPackage` | `Default` | `""` |
| `txtSolicitor` | `Default` | `If(IsBlank(varPaymentRecord), "", varPaymentRecord.'Assigned Solicitor')` |
| `txtNotes` | `Default` | `If(IsBlank(varPaymentRecord), "", varPaymentRecord.Notes)` |
| `ddLineFund` | `Default` | `If(varEditLineId = 0, "", LookUp(colFundLines, lineId = varEditLineId).fundLabel)` |
| `txtLineAmount` | `Default` | `If(varEditLineId = 0, "", Text(LookUp(colFundLines, lineId = varEditLineId).amount, "0.00"))` |
| `ddGiftType` | `Items` | `colGiftTypes` |
| `ddPaymentType` | `Items` | `colPaymentTypes` |
| `ddCampaign` | `Items` | `colCampaigns` |
| `ddAppeal` | `Items` | `colAppeals` |
| `ddFund` | `Items` | `colFunds` |
| `ddLineFund` | `Items` | `colFunds` |
| `btnSubmit` | `DisplayMode` | `DisplayMode.Edit` |
| `lblFundTotal` | `Text` | `"Total des lignes : " & Text(Sum(colFundLines, amount), "$#,##0.00") & If((IsMatch(Trim(txtGiftAmount.Text), "^\d+(\.\d{1,2})?$") && Value(txtGiftAmount.Text) > 0), " sur " & Text(Value(txtGiftAmount.Text), "$#,##0.00"), "")` |

### Validation messages

| Label | Text | Visible | Color |
|---|---|---|---|
| `lblErrLine` | `varLineError` | `tglSplitFunds.Value && !IsBlank(varLineError)` | `Color.Red` |
| `lblErrNewDonorName` | `"Enter the new donor’s name."` | `varTried && IsBlank(Trim(txtNewDonorName.Text))` | `Color.Red` |
| `lblErrGiftDate` | `"Choisissez la date du don."` | `varTried && IsBlank(dpGiftDate.SelectedDate)` | `Color.Red` |
| `lblErrGiftType` | `"Choisissez le type de don."` | `varTried && IsBlank(ddGiftType.Selected.code)` | `Color.Red` |
| `lblErrPaymentType` | `"Choisissez le mode de paiement."` | `varTried && IsBlank(ddPaymentType.Selected.code)` | `Color.Red` |
| `lblErrCampaign` | `"Choisissez une campagne."` | `varTried && IsBlank(ddCampaign.Selected.code)` | `Color.Red` |
| `lblErrAppeal` | `"Choisissez un appel."` | `varTried && IsBlank(ddAppeal.Selected.code)` | `Color.Red` |
| `lblErrFund` | `"Choisissez un fonds."` | `varTried && !(tglSplitFunds.Value \|\| !IsBlank(ddFund.Selected.code))` | `Color.Red` |
| `lblErrGiftAmount` | `"Montant supérieur à 0, par exemple 100.00."` | `varTried && !(IsMatch(Trim(txtGiftAmount.Text), "^\d+(\.\d{1,2})?$") && Value(txtGiftAmount.Text) > 0)` | `Color.Red` |
| `lblErrFees` | `"Frais : un nombre comme 2.50, sans dépasser le montant."` | `varTried && !(IsBlank(Trim(txtFees.Text)) \|\| (IsMatch(Trim(txtFees.Text), "^\d+(\.\d{1,2})?$") && (!(IsMatch(Trim(txtGiftAmount.Text), "^\d+(\.\d{1,2})?$") && Value(txtGiftAmount.Text) > 0) \|\| Value(txtFees.Text) <= Value(txtGiftAmount.Text))))` | `Color.Red` |
| `lblErrTaxReceipt` | `"Avantage et reçu : entre 0 $ et le montant du don."` | `varTried && tglTaxReceipt.Value && !((IsBlank(Trim(txtBenefitAmount.Text)) \|\| (IsMatch(Trim(txtBenefitAmount.Text), "^\d+(\.\d{1,2})?$") && Value(txtBenefitAmount.Text) >= 0 && (!(IsMatch(Trim(txtGiftAmount.Text), "^\d+(\.\d{1,2})?$") && Value(txtGiftAmount.Text) > 0) \|\| Value(txtBenefitAmount.Text) <= Value(txtGiftAmount.Text)))) && (IsMatch(Trim(txtReceiptAmount.Text), "^\d+(\.\d{1,2})?$") && Value(txtReceiptAmount.Text) >= 0 && (!(IsMatch(Trim(txtGiftAmount.Text), "^\d+(\.\d{1,2})?$") && Value(txtGiftAmount.Text) > 0) \|\| Value(txtReceiptAmount.Text) <= Value(txtGiftAmount.Text))))` | `Color.Red` |
| `lblErrFundLines` | `If(CountRows(colFundLines) = 0, "Ajoutez au moins une ligne de fonds.", "La somme des lignes doit égaler le montant du don.")` | `varTried && !(!tglSplitFunds.Value \|\| (CountRows(colFundLines) > 0 && (IsMatch(Trim(txtGiftAmount.Text), "^\d+(\.\d{1,2})?$") && Value(txtGiftAmount.Text) > 0) && Abs(Sum(colFundLines, amount) - Value(txtGiftAmount.Text)) < 0.005))` | `Color.Red` |
| `lblErrorConstituentId` | `If(IsBlank(Trim(txtConstituentId.Text)), "Saisissez l'ID (7 chiffres), puis cliquez sur Lookup.", "Cliquez sur Lookup pour vérifier le donateur.")` | `varTried && (rdoNewDonor.Selected.Value = "No") && !(varDonor.state = "found" && varDonor.lookupId = Trim(txtConstituentId.Text)) && IsBlank(varDonor.message)` | `Color.Red` |
| `lblDonorMsg` | `varDonor.message` | `(rdoNewDonor.Selected.Value = "No") && !IsBlank(varDonor.message)` | `Color.Red` |
| `lblConstituentName` | `varDonor.name` | `(rdoNewDonor.Selected.Value = "No") && varDonor.state = "found"` | `RGBA(33, 37, 41, 1)` |
| `lblSolicitor` | `If(varSolicitor.state = "found", varSolicitor.name, varSolicitor.message)` | `true` | `If(varSolicitor.state = "found", Color.Black, Color.Red)` |
| `lblErrSolicitor` | `"Check the solicitor's ID, or leave it empty."` | `varTried && !IsBlank(Trim(txtSolicitor.Text)) && varSolicitor.state <> "found"` | `Color.Red` |

`lblNetAmount.Text`:

```powerfx
If(!(IsMatch(Trim(txtGiftAmount.Text), "^\d+(\.\d{1,2})?$") && Value(txtGiftAmount.Text) > 0) || !(IsBlank(Trim(txtFees.Text)) || (IsMatch(Trim(txtFees.Text), "^\d+(\.\d{1,2})?$") && (!(IsMatch(Trim(txtGiftAmount.Text), "^\d+(\.\d{1,2})?$") && Value(txtGiftAmount.Text) > 0) || Value(txtFees.Text) <= Value(txtGiftAmount.Text)))), "",
    Text(Value(txtGiftAmount.Text) - Coalesce(Value(txtFees.Text), 0), "$#,##0.00"))
```

## 5. Tests

Run on 2026-11-10 in Jérémie's personal environment (#8 / PR #38), on
2026-11-14 in Yasmine's personal environment (#22), and on 2026-11-18 in
Yasmine's personal environment (#23). Test donor `1234567` (Camille Exemple);
`9999500` is the simulated NXT's error-500 ID.

| # | Steps | Expected | Result |
|---|---|---|---|
| 1 | Open, click Submit without touching anything | A red message under every required field; nothing saved | Pass |
| 2 | ID `12345`, Lookup | "Enter the 7-digit Constituent ID." | Pass |
| 3 | ID `9999500`, Lookup | "Donor lookup unavailable. Try again in a moment." | Pass |
| 4 | ID `7654321`, Lookup | "No donor with this Constituent ID. Check the number." | Pass |
| 5 | ID `1234567`, Lookup | "Camille Exemple" | Pass |
| 6 | Amount `100`, fees `150` | Fees message, no net amount | Pass |
| 7 | Amount `100`, fees `2.50` | Net amount $97.50, amount still 100 | Pass |
| 8 | Complete gift: 2026-10-15, One-time gift, Personal Cheque, AC-2026-2027 / WEB-2026 / HA-CARD0110 | Popup `SUB-00022`; submission with fees 2.5, net 97.5; fund line `HA-CARD0110`, $100, `Submission` = 22, `Submission Reference` = `SUB-00022` | Pass |
| 9 | Fermer et réinitialiser | Every field back on its placeholder, donor cleared | Pass |
| 10 | Open with `?paymentId=12` | Donor looked up automatically; 2026-11-05, 100.00, Personal Cheque; gift type empty (the payment has none) | Pass |
| 11 | Code it (One-time gift, AC-2025-2026 / EV-GOLF-2026 / HA-GENE0001), Submit | `SUB-00023` with `Payment Reference` = `PAY-00012`; payment 12 has `Submission Reference` = `SUB-00023`; fund line linked | Pass |
| 12 | Open with `?paymentId=12` again | Warning "This payment is already coded as SUB-00023…" | Pass |
| 13 | Open with `?paymentId=999` | Warning "Payment 999 is not in the Payment Log…" | Pass |
| 14 | $100 gift, split: HA-CARD0110 $60 + HA-ONCO0220 $30, Submit | Blocked: "La somme des lignes doit égaler le montant du don.", total $90.00 sur $100.00 | Pass |
| 15 | Edit icon on the $30 line, amount 40, "Mettre à jour la ligne" | Line loaded into the inputs, then $40.00; total $100.00 sur $100.00 | Pass |
| 16 | Add HA-CARD0110 again | "Ce fonds est déjà dans la répartition : modifiez sa ligne." | Pass |
| 17 | Add HA-PEDI0330 $10, then its delete icon | Total $110.00 with the message, then back to two lines and $100.00 | Pass |
| 18 | New donor, Submit without a name | ID and Lookup off; "Saisissez le nom du nouveau donateur."; no Constituent ID message | Pass |
| 19 | New donor "Julie Gagnon", 2026-11-02, Cash, $45, EV-2026 / EV-COURSE-2026 / HA-GENE0001 | `SUB-00026` with no Constituent ID, name "Julie Gagnon"; fund line $45 | Pass |
| 20 | Solicitor `12345` (not 7 digits), Check | `"Enter the solicitor's 7-digit Constituent ID."` in red; Submit blocked with `"Check the solicitor's ID, or leave it empty."` | Pass |
| 21 | Solicitor `7654321` (not found), Check | `"No solicitor with this ID."` in red; Submit blocked | Pass |
| 22 | Solicitor `1234567` (Camille Exemple, `fundraiser_status = "None"`), Check | `"Camille Exemple is not an active solicitor."` in red; Submit blocked | Pass |
| 23 | Edit `txtSolicitor` after Check without re-checking | `varSolicitor` cleared on `OnChange`; Submit blocked until re-checked or cleared | Pass |
| 24 | Complete gift with `Package` = `PKG-GALA-2026`, `Notes` (2 lines), `Solicitor` empty, Submit (`SUB-00035`, `SUB-00036`, `SUB-00037`) | Saved to `MCHF Submissions` with `Package`, multiline `Notes` and empty `Assigned Solicitor` | Pass |
| 25 | Fermer et réinitialiser after #24 | `txtPackage`, `txtSolicitor`, `txtNotes`, fund lines and split toggle all cleared | Pass |
| 26 | Open with a Payment Log item (`Assigned Solicitor = 2012008`, `Notes = "Cheque received at the gala"`) via `?paymentId=<ID>`, Check, Submit | Donor (`varDonor`), `2012008` in `txtSolicitor` and `"Cheque received at the gala"` in `txtNotes` shown once on load; Check validates active solicitor, Submit persists both | Blocked on 7-digit NXT rollout (prefill verified; `2012008` currently returns not found) |
| 27 | Solicitor `9999500`, Check | `"Solicitor check unavailable. Try again in a moment."` in red; Submit blocked | Pass |
| 28 | Solicitor `2012066` (Andrée Savard, inactive record), Check | Refused as inactive (`"Andrée Savard is not an active solicitor."`); Submit blocked | Blocked on 7-digit NXT rollout (currently returns `"No solicitor with this ID."`) |
| 29 | Solicitor `2012008` (Raphaël Michaud, active), Check + `Package` + `Notes` all filled, Submit | `"Raphaël Michaud"` in black; Submit accepted and `2012008`, `Package` and `Notes` saved to `MCHF Submissions` | Blocked on 7-digit NXT rollout (currently returns `"No solicitor with this ID."`) |
| 30 | Tax receipt toggle on No (default) | `txtBenefitAmount` and `txtReceiptAmount` hidden | Pass |
| 31 | Gift `$100`, toggle Tax receipt to Yes, Benefit `$25`, then override Receipt to `$80` and Submit | `txtReceiptAmount` defaults to `75.00`; changing it to `80` is accepted and saved with `ReceiptAmount = 80` and `BenefitAmount = 25` | Pass |
| 32 | Gift `$100`, Benefit `$2`, Submit (`SUB-00047`) | Saved to `MCHF Submissions` with `TaxReceipt = true`, `BenefitAmount = 2`, `ReceiptAmount = 98`, and `NetAmount` unchanged (`100`) | Pass |
| 33 | Gift `$100`, Benefit `$120` (or Receipt `> $100` / `< $0`), Submit | Blocked: `"Avantage et reçu : entre 0 $ et le montant du don."` in red | Pass |
| 34 | Toggle Tax receipt to Yes, enter Benefit `$20`, switch back to No, Submit | Both inputs hidden and reset; saved with `TaxReceipt = false`, `BenefitAmount` and `ReceiptAmount` empty | Pass |

Tests 35–37 were run in Alexandre’s personal environment for #25, with all
other required gift fields completed.

| # | Steps | Expected | Result |
|---|---|---|---|
| 35 | Select Yes, enter `Jean Nouveau`, submit a $20 gift | `SUB-00035` saved with `New Donor = true`, the typed name, and both constituent IDs empty | Pass |
| 36 | Select Yes, leave the donor name empty, submit | Name error appears and Submit is blocked | Pass |
| 37 | Switch to No, enter a valid seven-digit ID, submit before clicking Lookup | Lookup is required again and Submit is blocked | Pass |

Same-donor tests in Alexandre’s personal environment (#26):

| # | Steps | Result |
|---|---|---|
| 38 | Look up `1234567` on a blank form | Six Unclaimed payments appeared. The existing Coded payment `PAY-00016` was excluded. |
| 39 | Claim `PAY-00012` from the panel and submit the $100 gift | The payment was Claimed by Alexandre. `SUB-00036` stored `PAY-00012`, and the payment stored `SUB-00036`. |
| 40 | With the panel open, set `PAY-00013` to Claimed by Test in a second tab, then try to claim it from the panel | The second claim was refused. Test remained the claimant and the submission reference stayed empty. |

Both test payments were restored to Unclaimed and their claim details and submission references were cleared. Additional preview checks: dismiss the panel, look up `7654321`, and open the form with `?paymentId=12`. The last two paths should leave the panel hidden.

Not run: `9999504` (timeout; same `IfError` branch as tests 3 and 27, after a 10 s
wait). Tests 26 (Check + Submit), 28 (`2012066`) and 29 (`2012008`) are ready and
waiting for Jérémie's 7-digit NXT constituent migration. App checker: no errors.

### Pledge payment tests (#53)

In Alexandre’s personal environment:

| Test | Result |
|---|---|
| Look up donor `1234567` | Open pledges `51308` ($300) and `51309` ($200) show their balances. Fully paid pledge `51311` is absent. |
| Search for pledge owner `9999002` | Third-party pledge `51310` ($500) appears and the gift’s donor stays `1234567`. |
| Add an amount above the balance or add a duplicate pledge | The form refuses the line and shows the corresponding message. |
| Edit and delete a pledge line | Editing updates the existing payment amount. Deleting removes the line. |
| Choose Yes with gift type One-time gift, or submit lines that do not total the Gift Amount | Submit is blocked with the corresponding message. |
| Submit a $150 pledge payment split between two pledges | `SUB-00038` has two linked rows, $100 on `51308` and $50 on `51309`. Closing the success popup clears the pledge section and restores No. |
| Add a line, switch to No, then submit a valid one-time gift | `SUB-00039` has `Pledge Payment` set to No and zero linked pledge rows. |

## 6. Where it lives

| What | Where |
|---|---|
| App | **MCHF - Coding Form**, Équipe (play: Applications › MCHF - Coding Form) |
| Lists | `MCHF Submissions`, `MCHF Fund Lines`, `MCHF Payment Types`, `MCHF Gift Types`, `MCHF - Payment Log`, `MCHF - Pledge Payments` |
| Test flow | `MCHF - Create test pledges`, manual flow for simulated RE NXT |
| Connector | RE NXT (simulated), `RENXT`, with the connection of the person using the app |
| Export | `lab/solutions/mchf-coding-form.solution.json` |

## 7. Limits and remaining work

- **Abandoned claims (#26).** A payment claimed from the panel stays Claimed if the fundraiser leaves before submitting. Gift Processing can release it manually. Automatic release is an open question for Marc.
- **Fresh environment.** In a fresh environment, make the required lists available, including `MCHF - Pledge Payments` and the `Pledge Payment` field on `MCHF Submissions`. Populate the active gift types and payment types before testing Submit. To test the panel, add an Unclaimed payment for a donor the lookup can find.

- **New donor form URLs (#25).** Both buttons use placeholder addresses. When
  Marc provides the Foundation URLs, update `varNewIndividualUrl` and
  `varNewOrganizationUrl` in `Screen1.OnVisible`.
- **`Constituent ID` must be optional on the list** in every environment the app runs
  in, otherwise a new donor's submission is refused ("The submission was not
  saved"). A change request only adds missing columns; it does not change an
  existing one.
- **Package is free text in the lab.** `txtPackage` is a single-line text input in
  the lab; the real RE NXT connector provides *List packages* filtered by appeal,
  to be wired in Sprint 4.
- **Only the solicitor's Constituent ID is saved.** `MCHF Submissions.Assigned Solicitor`
  stores the 7-digit `lookup_id` (`Trim(txtSolicitor.Text)`), not the solicitor's
  display name.
- **Tax receipt default value (open question for Marc).** `tglTaxReceipt` currently
  defaults to `false` (No). Should the form force an explicit choice (no default
  selected) before Submit?
- **Coding a payment twice** is warned about, not blocked.
- **No transaction.** If a write after step 1 fails, the submission already exists; the
  message names it and the rest is fixed by hand. SharePoint has no transactions
  either.
- **Rebuild in Power Apps (Sprint 4).** The formulas use standard functions only.
  Check there: `Param` in a published app, the Dropdown's first-item behaviour (same
  as classic Power Apps), `Patch` returning the saved record (same with SharePoint),
  and the real connector's operation name (`SearchConstituent`, not
  `SearchConstituents`, see `DIFFERENCES.md` of the lab).




