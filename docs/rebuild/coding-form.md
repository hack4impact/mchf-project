# Coding Form: Simple Gift

Related issues: [#8](https://github.com/hack4impact/mchf-project/issues/8) (with follow-ups from PR #38), [#22](https://github.com/hack4impact/mchf-project/issues/22) (Package, Assigned Solicitor and Notes)
App: **MCHF - Coding Form** (Équipe)
Lab export: `lab/solutions/mchf-coding-form.solution.json` (the app and the five lists it uses, no items)

A fundraiser enters one simple gift: the donor (or a new donor), the gift, its
coding (campaign, appeal, one fund or a split across several funds, optional package,
assigned solicitor and notes). The form saves it as a submission with the status
`Submitted`, gives it a reference `SUB-00001`, writes its fund lines, and links it
to the payment it codes when it was opened from the Payment Log.

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
| 4 | `MCHF - Payment Log` | Only when opened with `paymentId`: `Submission Reference` = the SUB number |

Then a popup shows the SUB number.

**Submission fields:**

| Column | Value |
|---|---|
| `Constituent ID` | The ID found by the lookup; empty for a new donor (Gift Processing adds it before approving) |
| `Constituent Name` | The name returned by NXT, or the new donor's name as typed |
| `Constituent System ID` | The NXT system ID returned by the lookup, empty for a new donor |
| `Payment Type`, `Gift Type` | `Title` of the chosen `MCHF Payment Types` / `MCHF Gift Types` item |
| `Gift Date` | The date picker |
| `Gift Amount` | Typed amount |
| `Fees` | Typed fees, `0` when empty |
| `Net Amount` | `Gift Amount - Fees` |
| `Campaign`, `Appeal` | NXT `lookup_id` of the chosen item (e.g. `AC-2025-2026`, `EV-GOLF-2026`) |
| `Fund` | Primary fund: the chosen fund, or with a split the line with the largest amount (e.g. `HA-GENE0001`) |
| `Package` | Optional free-text package code (`Trim(txtPackage.Text)`), empty when omitted |
| `Assigned Solicitor` | Optional 7-digit solicitor Constituent ID (`Trim(txtSolicitor.Text)`), validated as an active fundraiser in RE NXT; prefilled from the payment when present, never from the signed-in user |
| `Notes` | Optional multiline gift notes (`txtNotes.Text`), prefilled from `varPaymentRecord.Notes` when present |
| `Payment Reference` | The payment's `Reference` (e.g. `PAY-00012`), when there is one |

## 2. Screen

`Screen1`, tablet 1366 × 768. Labels are in French, as built; the donor and
solicitor lookup messages are the English ones of the #2 and #22 contracts.

- **Left column:** `txtConstituentId` + `btnSearchDonor` ("Lookup"); under them one
  line that shows the donor's name (`lblConstituentName`), a lookup message
  (`lblDonorMsg`) or the validation message (`lblErrorConstituentId`); the toggle
  `tglNewDonor`, then for a new donor `btnNewDonorLink` and `txtNewDonorName`;
  `txtGiftAmount`, `txtFees`, `lblNetAmount`; `btnSubmit`.
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
- **New donor** (PRD acceptance #19 and the decision on new donors). "Nouveau donateur ?"
  turns off the ID field and Lookup, shows the link to the new-donor form and a
  required name field. The submission is saved without a Constituent ID; Gift
  Processing adds it before approving. This needs `Constituent ID` optional at list
  level (data dictionary).
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
  `txtPackage`, `txtSolicitor` and `txtNotes`), `varDonor`, `varSolicitor`, the
  fund lines and `varPaymentRecord`, so the next gift cannot be linked to the
  payment that was just coded.

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
                {state: "error", systemId: "", lookupId: "", name: "", message: "Donor lookup unavailable. Try again in a moment."})))))
```

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
            {state: "error", systemId: "", lookupId: "", name: "", message: "Donor lookup unavailable. Try again in a moment."}))))
```

`txtConstituentId.OnChange`:

```powerfx
Set(varDonor, Blank())
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

### `btnSubmit.OnSelect`

```powerfx
UpdateContext({varTried: true});
If(
    (tglNewDonor.Value || (varDonor.state = "found" && varDonor.lookupId = Trim(txtConstituentId.Text))) &&
    (!tglNewDonor.Value || !IsBlank(Trim(txtNewDonorName.Text))) &&
    !IsBlank(dpGiftDate.SelectedDate) &&
    !IsBlank(ddGiftType.Selected.code) &&
    !IsBlank(ddPaymentType.Selected.code) &&
    !IsBlank(ddCampaign.Selected.code) &&
    !IsBlank(ddAppeal.Selected.code) &&
    (tglSplitFunds.Value || !IsBlank(ddFund.Selected.code)) &&
    (IsMatch(Trim(txtGiftAmount.Text), "^\d+(\.\d{1,2})?$") && Value(txtGiftAmount.Text) > 0) &&
    (IsBlank(Trim(txtFees.Text)) || (IsMatch(Trim(txtFees.Text), "^\d+(\.\d{1,2})?$") && (!(IsMatch(Trim(txtGiftAmount.Text), "^\d+(\.\d{1,2})?$") && Value(txtGiftAmount.Text) > 0) || Value(txtFees.Text) <= Value(txtGiftAmount.Text)))) &&
    (!tglSplitFunds.Value || (CountRows(colFundLines) > 0 && (IsMatch(Trim(txtGiftAmount.Text), "^\d+(\.\d{1,2})?$") && Value(txtGiftAmount.Text) > 0) && Abs(Sum(colFundLines, amount) - Value(txtGiftAmount.Text)) < 0.005)) &&
    (IsBlank(Trim(txtSolicitor.Text)) || varSolicitor.state = "found"),
    // 1. Save with a temporary Title, 2. set the reference SUB-00001 from the new ID (#1 data dictionary),
    // 3. the fund lines (the split, or the fund chosen with the whole amount), 4. link the payment.
    // Stops at the first failure. The submission's Fund is the primary fund: the largest line.
    With({lines: If(tglSplitFunds.Value, colFundLines,
            Table({lineId: 1, fund: ddFund.Selected.code, fundLabel: ddFund.Selected.Value, amount: Value(txtGiftAmount.Text)}))},
    With({created: Patch('MCHF Submissions', Defaults('MCHF Submissions'), {
            Title: "NEW",
            'Constituent ID': If(tglNewDonor.Value, Blank(), varDonor.lookupId),
            'Constituent Name': If(tglNewDonor.Value, Trim(txtNewDonorName.Text), varDonor.name),
            'Constituent System ID': If(tglNewDonor.Value, "", varDonor.systemId),
            'Payment Type': ddPaymentType.Selected.code,
            'Gift Date': dpGiftDate.SelectedDate,
            'Gift Type': ddGiftType.Selected.code,
            'Gift Amount': Value(txtGiftAmount.Text),
            Fees: Coalesce(Value(txtFees.Text), 0),
            'Net Amount': Value(txtGiftAmount.Text) - Coalesce(Value(txtFees.Text), 0),
            Campaign: ddCampaign.Selected.code,
            Appeal: ddAppeal.Selected.code,
            Fund: First(SortByColumns(lines, "amount", SortOrder.Descending)).fund,
            Package: Trim(txtPackage.Text),
            'Assigned Solicitor': Trim(txtSolicitor.Text),
            Notes: txtNotes.Text,
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
Reset(txtConstituentId); Reset(tglNewDonor); Reset(txtNewDonorName); Reset(txtGiftAmount); Reset(txtFees); Reset(dpGiftDate);
Reset(ddGiftType); Reset(ddPaymentType); Reset(ddCampaign); Reset(ddAppeal); Reset(ddFund);
Reset(txtPackage); Reset(txtSolicitor); Reset(txtNotes);
Reset(tglSplitFunds); Reset(ddLineFund); Reset(txtLineAmount)
```

### Defaults

| Control | Property | Formula |
|---|---|---|
| `txtConstituentId` | `Default` | `If(IsBlank(varPaymentRecord), "", varPaymentRecord.'Constituent ID')` |
| `txtConstituentId` | `DisplayMode` | `If(tglNewDonor.Value, DisplayMode.Disabled, DisplayMode.Edit)` |
| `txtNewDonorName` | `Visible` | `tglNewDonor.Value` |
| `txtGiftAmount` | `Default` | `If(IsBlank(varPaymentRecord), "", Text(varPaymentRecord.'Gift Amount', "0.00"))` |
| `txtFees` | `Default` | `"0.00"` |
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
| `lblErrNewDonorName` | `"Saisissez le nom du nouveau donateur."` | `varTried && !(!tglNewDonor.Value \|\| !IsBlank(Trim(txtNewDonorName.Text)))` | `Color.Red` |
| `lblErrGiftDate` | `"Choisissez la date du don."` | `varTried && IsBlank(dpGiftDate.SelectedDate)` | `Color.Red` |
| `lblErrGiftType` | `"Choisissez le type de don."` | `varTried && IsBlank(ddGiftType.Selected.code)` | `Color.Red` |
| `lblErrPaymentType` | `"Choisissez le mode de paiement."` | `varTried && IsBlank(ddPaymentType.Selected.code)` | `Color.Red` |
| `lblErrCampaign` | `"Choisissez une campagne."` | `varTried && IsBlank(ddCampaign.Selected.code)` | `Color.Red` |
| `lblErrAppeal` | `"Choisissez un appel."` | `varTried && IsBlank(ddAppeal.Selected.code)` | `Color.Red` |
| `lblErrFund` | `"Choisissez un fonds."` | `varTried && !(tglSplitFunds.Value \|\| !IsBlank(ddFund.Selected.code))` | `Color.Red` |
| `lblErrGiftAmount` | `"Montant supérieur à 0, par exemple 100.00."` | `varTried && !(IsMatch(Trim(txtGiftAmount.Text), "^\d+(\.\d{1,2})?$") && Value(txtGiftAmount.Text) > 0)` | `Color.Red` |
| `lblErrFees` | `"Frais : un nombre comme 2.50, sans dépasser le montant."` | `varTried && !(IsBlank(Trim(txtFees.Text)) \|\| (IsMatch(Trim(txtFees.Text), "^\d+(\.\d{1,2})?$") && (!(IsMatch(Trim(txtGiftAmount.Text), "^\d+(\.\d{1,2})?$") && Value(txtGiftAmount.Text) > 0) \|\| Value(txtFees.Text) <= Value(txtGiftAmount.Text))))` | `Color.Red` |
| `lblErrFundLines` | `If(CountRows(colFundLines) = 0, "Ajoutez au moins une ligne de fonds.", "La somme des lignes doit égaler le montant du don.")` | `varTried && !(!tglSplitFunds.Value \|\| (CountRows(colFundLines) > 0 && (IsMatch(Trim(txtGiftAmount.Text), "^\d+(\.\d{1,2})?$") && Value(txtGiftAmount.Text) > 0) && Abs(Sum(colFundLines, amount) - Value(txtGiftAmount.Text)) < 0.005))` | `Color.Red` |
| `lblErrorConstituentId` | `If(IsBlank(Trim(txtConstituentId.Text)), "Saisissez l'ID (7 chiffres), puis cliquez sur Lookup.", "Cliquez sur Lookup pour vérifier le donateur.")` | `varTried && !tglNewDonor.Value && !(varDonor.state = "found" && varDonor.lookupId = Trim(txtConstituentId.Text)) && IsBlank(varDonor.message)` | `Color.Red` |
| `lblDonorMsg` | `varDonor.message` | `!tglNewDonor.Value && !IsBlank(varDonor.message)` | `Color.Red` |
| `lblConstituentName` | `varDonor.name` | `!tglNewDonor.Value && varDonor.state = "found"` | `RGBA(33, 37, 41, 1)` |
| `lblSolicitor` | `If(varSolicitor.state = "found", varSolicitor.name, varSolicitor.message)` | `true` | `If(varSolicitor.state = "found", Color.Black, Color.Red)` |
| `lblErrSolicitor` | `"Check the solicitor's ID, or leave it empty."` | `varTried && !IsBlank(Trim(txtSolicitor.Text)) && varSolicitor.state <> "found"` | `Color.Red` |

`lblNetAmount.Text`:

```powerfx
If(!(IsMatch(Trim(txtGiftAmount.Text), "^\d+(\.\d{1,2})?$") && Value(txtGiftAmount.Text) > 0) || !(IsBlank(Trim(txtFees.Text)) || (IsMatch(Trim(txtFees.Text), "^\d+(\.\d{1,2})?$") && (!(IsMatch(Trim(txtGiftAmount.Text), "^\d+(\.\d{1,2})?$") && Value(txtGiftAmount.Text) > 0) || Value(txtFees.Text) <= Value(txtGiftAmount.Text)))), "",
    Text(Value(txtGiftAmount.Text) - Coalesce(Value(txtFees.Text), 0), "$#,##0.00"))
```

## 5. Tests

Run on 2026-11-10 in Jérémie's personal environment (#8 / PR #38) and on
2026-11-14 in Yasmine's personal environment (#22). Test donor `1234567`
(Camille Exemple); `9999500` is the simulated NXT's error-500 ID.

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
| 26 | Open with a Payment Log item having `Assigned Solicitor` and `Notes` (`?paymentId=<ID>`) | `txtSolicitor` prefilled with the payment's solicitor ID (never the signed-in user) and `txtNotes` prefilled with the payment's notes; Submit blocked until Check validates the solicitor | Pass |

Not run: `9999504` (timeout; same `IfError` branch as test 3, after a 10 s
wait), and active solicitor positive lookup with ID `2012008` (currently returns
not found on the simulated NXT instance; reported to Jérémie). App checker: no errors.

## 6. Where it lives

| What | Where |
|---|---|
| App | **MCHF - Coding Form**, Équipe (play: Applications › MCHF - Coding Form) |
| Lists | `MCHF Submissions`, `MCHF Fund Lines`, `MCHF Payment Types`, `MCHF Gift Types`, `MCHF - Payment Log` |
| Connector | RE NXT (simulated), `RENXT`, with the connection of the person using the app |
| Export | `lab/solutions/mchf-coding-form.solution.json` |

## 7. Limits and remaining work

- **New donor link.** `btnNewDonorLink` still opens `https://renxt.blackbaud.com`.
  The PRD decided to link to the Foundation's existing new-donor form, but the two
  forms have not been obtained yet (PRD open questions; #25). Replace the address in
  `btnNewDonorLink.OnSelect` once the link is known.
- **`Constituent ID` must be optional on the list** in every environment the app runs
  in, otherwise a new donor's submission is refused ("The submission was not
  saved"). A change request only adds missing columns; it does not change an
  existing one.
- **Coding a payment twice** is warned about, not blocked.
- **No transaction.** If step 2, 3 or 4 fails, the submission already exists; the
  message names it and the rest is fixed by hand. SharePoint has no transactions
  either.
- **Rebuild in Power Apps (Sprint 4).** The formulas use standard functions only.
  Check there: `Param` in a published app, the Dropdown's first-item behaviour (same
  as classic Power Apps), `Patch` returning the saved record (same with SharePoint),
  and the real connector's operation name (`SearchConstituent`, not
  `SearchConstituents`, see `DIFFERENCES.md` of the lab).