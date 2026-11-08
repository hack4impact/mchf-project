# Rebuild Guide: Coding Form (Simple Gift - Issue #8)

## 1. Screens & Layout
- **Écran principal** : `Screen1` (Format Tablette 1366 × 768)[cite: 46].

### Entête
- `rectHeaderBg` (Rectangle) : Bandeau supérieur vert foncé.
- `lblHeader` (Label) : Titre de l'application (« MCHF - Coding Form »).

### Colonne de gauche (Identification Donateur & Montants)
- **Donateur** :
  - `lblDonatorID` (Label) : Libellé « ID Donateur ».
  - `txtConstituentId` (Text Input) : Champ de saisie pour l'identifiant donateur (7 chiffres)[cite: 52].
  - `btnSearchDonor` (Button) : Bouton « Lookup » déclenchant la recherche via l'API RENXT[cite: 49].
  - `lblConstituentName` (Label) : Affichage en gras du nom du donateur retourné, positionné à droite du bouton Lookup.
  - `lblDonorMsg` (Label) : Affichage des messages d'erreur ou d'alerte en rouge, situé au-dessus de la bascule.
  - `lblErrorConstituentId` (Label) : Message de validation rouge sous le champ ID donateur.
  - `tglNewDonor` (Toggle) : Bascule « Nouveau donateur ? ».
  - `lblNewDonator` (Label) : Texte descriptif associé à la bascule.
  - `btnNewDonorLink` (Button / Link) : Action d'aide à la création de fiche donateur.
- **Montants & Calculs** :
  - `lblGiftAmount` (Label) : Libellé « Montant du don ($) ».
  - `txtGiftAmount` (Text Input) : Saisie du montant brut du don.
  - `lblFees` (Label) : Libellé « Frais ($) ».
  - `txtFees` (Text Input) : Saisie des frais de traitement bancaire.
  - `lblNetAmountText` (Label) : Libellé statique « Montant net : ».
  - `lblNetAmount` (Label) : Affichage dynamique du montant net calculé.

### Colonne de droite (Détails & Imputation comptable)
- `lblGiftDate` (Label) : Libellé « Date du don ».
- `dpGiftDate` (Date Picker) : Sélection de la date de réception du don[cite: 47].
- `lblGiftType` (Label) : Libellé « Type de don ».
- `ddGiftType` (Dropdown) : Menu déroulant pour le type de don[cite: 47].
- `lblPaymentType` (Label) : Libellé « Mode de paiement ».
- `ddPaymentType` (Dropdown) : Menu déroulant du mode de règlement[cite: 47].
- `lblCampaign` (Label) : Libellé « Campagne ».
- `ddCampaign` (Dropdown) : Menu déroulant alimenté dynamiquement via RENXT[cite: 47, 49].
- `lblAppeal` (Label) : Libellé « Appel ».
- `ddAppeal` (Dropdown) : Menu déroulant alimenté dynamiquement via RENXT[cite: 47, 49].
- `lblFunds` (Label) : Libellé « Fonds ».
- `ddFund` (Dropdown) : Menu déroulant d'imputation comptable via RENXT[cite: 47, 49].
- `btnSubmit` (Button) : Bouton d'enregistrement et de soumission finale[cite: 47].

### Fenêtre modale de confirmation (Succès)
- `rectOverlay` (Rectangle) : Fond grisé semi-transparent couvrant l'écran[cite: 47].
- `rectModalCard` (Rectangle) : Carte centrale blanche du pop-up[cite: 47].
- `lblSuccessTitle` (Label) : Titre de confirmation (« Soumission réussie »)[cite: 47].
- `lblSuccessRef` (Label) : Affichage de la référence unique générée (`varSubmissionRef`)[cite: 47].
- `btnClosePopup` (Button) : Bouton pour fermer la fenêtre modale[cite: 47].

---

## 2. Business Rules & Logic
- **Chargement dynamique RENXT (Guide Dev 2 - Section 2.2)** : Les listes `Campagne`, `Appel` et `Fonds` sont chargées dynamiquement au démarrage depuis l'API RE NXT simulée via `ClearCollect`[cite: 49]. Application du motif standard `{Value: lookup_id & " - " & description, code: lookup_id}` : l'interface affiche `Value` pour l'utilisateur et le système enregistre `code` dans SharePoint[cite: 49].
- **Gestion et validation du donateur (Guide Dev 2 - Section 2.3)** : Validation par expression régulière (`^\d{7}$`)[cite: 52]. Déclenchement de `RENXT.SearchConstituents(txtConstituentId.Text, {search_field: "lookup_id"})` et gestion déterministe des quatre états : `found`, `notfound`, `invalid` et `error`[cite: 49, 52]. Si `tglNewDonor.Value` est activé, l'enregistrement force `Constituent ID` à `"NEW"` et `Constituent Name` à `"Nouveau donateur"`.
- **Calcul dynamique du montant net (Guide Dev 2 - Section 2.4)** : Formule : `Value(txtGiftAmount.Text) - Value(txtFees.Text)`. Le montant brut reste intact dans `txtGiftAmount.Text`.
- **Validation bloquante (Guide Dev 2 - Section 2.4)** : Conditionnée par la variable `varTried`. Si un champ requis est manquant ou si le donateur n'est pas validé, la soumission est bloquée, un toast d'erreur est émis et les messages d'avertissement contextuels s'affichent sous les champs[cite: 48].
- **Liaison bidirectionnelle Payment Log & Submissions (Guide Dev 2 - Section 2.5 et Section 3)** : Lecture du paramètre d'URL `Param("paymentId")` pour préremplir le donateur, le montant et la date[cite: 48]. La soumission génère une référence unique formatée `SUB-YYYYMMDD-HHMMSS` inscrite dans `'Payment Reference'`[cite: 48]. Rétro-écriture immédiate dans la liste `MCHF - Payment Log` via `Patch` pour alimenter le champ `'Submission Reference'` avec cette clé unique[cite: 48].

---

## 3. PowerFX Formulas

### Écran principal (`Screen1`)
- **`Screen1.OnVisible`** :
```powerfx
// 1. Détection du paiement lié via paramètre d'URL
Set(varPaymentIdParam, Param("paymentId"));
Set(
    varPaymentRecord, 
    If(
        !IsBlank(varPaymentIdParam), 
        LookUp('MCHF - Payment Log', ID = Value(varPaymentIdParam) || Title = varPaymentIdParam), 
        Blank()
    )
);

// 2. Initialisation des types de paiement et types de don
ClearCollect(
    colPaymentTypes, 
    Table(
        {Value: "Personal Cheque"},
        {Value: "Credit Card"},
        {Value: "Cash"},
        {Value: "Direct Debit"},
        {Value: "E-Transfer"}
    )
);

ClearCollect(
    colGiftTypes, 
    Table(
        {Value: "One-time gift"},
        {Value: "Pledge"},
        {Value: "Pledge Payment"}
    )
);

// 3. Chargement dynamique des listes RENXT (motif {Value: ..., code: ...})
IfError(
    ClearCollect(colCampaigns, ForAll(RENXT.ListCampaigns().value, {Value: ThisRecord.lookup_id & " - " & ThisRecord.description, code: ThisRecord.lookup_id}));
    ClearCollect(colAppeals, ForAll(RENXT.ListAppeals().value, {Value: ThisRecord.lookup_id & " - " & ThisRecord.description, code: ThisRecord.lookup_id}));
    ClearCollect(colFunds, ForAll(RENXT.ListFunds().value, {Value: ThisRecord.lookup_id & " - " & ThisRecord.description, code: ThisRecord.lookup_id})),
    Notify("NXT lists are unavailable. Reopen the form to retry.", NotificationType.Error)
);

// 4. Initialisation des variables d'état du donateur et de l'interface
UpdateContext({
    varDonorState: "",
    varDonorSystemId: "",
    varDonorName: "",
    varDonorMessage: "",
    varTried: false,
    varSubmissionRef: "",
    varShowSuccessPopup: false
});
```

---

### Contrôles Donateur

- **`txtConstituentId.Default`** :
```powerfx
If(!IsBlank(varPaymentRecord), varPaymentRecord.'Constituent ID', "")
```

- **`txtConstituentId.OnChange`** :
```powerfx
UpdateContext({
    varDonorState: "",
    varDonorSystemId: "",
    varDonorName: "",
    varDonorMessage: ""
})
```

- **`btnSearchDonor.OnSelect`** :
```powerfx
If(
    !IsMatch(txtConstituentId.Text, "^\d{7}$"),
    UpdateContext({
        varDonorState: "invalid",
        varDonorSystemId: "",
        varDonorName: "",
        varDonorMessage: "Enter the 7-digit Constituent ID."
    }),
    With(
        {res: IfError(RENXT.SearchConstituents(txtConstituentId.Text, {search_field: "lookup_id"}).value, Table())},
        Switch(
            CountRows(res),
            0, UpdateContext({
                varDonorState: "notfound",
                varDonorSystemId: "",
                varDonorName: "",
                varDonorMessage: "No donor with this Constituent ID. Check the number."
            }),
            1, UpdateContext({
                varDonorState: "found",
                varDonorSystemId: Text(First(res).id),
                varDonorName: First(res).name,
                varDonorMessage: ""
            }),
            UpdateContext({
                varDonorState: "error",
                varDonorSystemId: "",
                varDonorName: "",
                varDonorMessage: "Several donors match. Contact Gift Processing."
            })
        )
    )
)
```

- **`lblConstituentName.Text`** :
```powerfx
varDonorName
```

- **`lblConstituentName.Visible`** :
```powerfx
!IsBlank(varDonorName) && !tglNewDonor.Value
```

- **`lblDonorMsg.Text`** :
```powerfx
varDonorMessage
```

- **`lblDonorMsg.Color`** :
```powerfx
Color.Red
```

- **`lblDonorMsg.Visible`** :
```powerfx
!IsBlank(varDonorMessage) && !tglNewDonor.Value
```

- **`lblErrorConstituentId.Text`** :
```powerfx
"Constituant requis (recherche obligatoire)"
```

- **`lblErrorConstituentId.Color`** :
```powerfx
Color.Red
```

- **`lblErrorConstituentId.Visible`** :
```powerfx
varTried && !tglNewDonor.Value && varDonorState <> "found"
```

---

### Contrôles Montants & Calculs

- **`txtGiftAmount.Default`** :
```powerfx
If(!IsBlank(varPaymentRecord), Text(varPaymentRecord.'Gift Amount'), "100.00")
```

- **`txtFees.Default`** :
```powerfx
"0.00"
```

- **`lblNetAmount.Text`** :
```powerfx
Text(
    Value(txtGiftAmount.Text) - Value(txtFees.Text),
    "$#,##0.00"
)
```

---

### Contrôles Imputation & Menus Déroulants

- **`dpGiftDate.DefaultDate`** :
```powerfx
If(!IsBlank(varPaymentRecord) && !IsBlank(varPaymentRecord.'Gift Date'), varPaymentRecord.'Gift Date', Today())
```

- **`ddPaymentType.Items`** :
```powerfx
colPaymentTypes
```

- **`ddGiftType.Items`** :
```powerfx
colGiftTypes
```

- **`ddCampaign.Items`** :
```powerfx
colCampaigns
```

- **`ddAppeal.Items`** :
```powerfx
colAppeals
```

- **`ddFund.Items`** :
```powerfx
colFunds
```

---

### Bouton de Soumission (`btnSubmit`)

- **`btnSubmit.OnSelect`** :
```powerfx
UpdateContext({ varTried: true });

If(
    // Contrôle des conditions obligatoires
    (tglNewDonor.Value || varDonorState = "found") && 
    !IsBlank(ddPaymentType.Selected.Value) && 
    !IsBlank(ddGiftType.Selected.Value) && 
    !IsBlank(dpGiftDate.SelectedDate) && 
    IsMatch(txtGiftAmount.Text, "^\d+(\.\d{1,2})?$") && 
    Value(txtGiftAmount.Text) > 0 && 
    !IsBlank(ddCampaign.Selected.code) && 
    !IsBlank(ddAppeal.Selected.code) && 
    !IsBlank(ddFund.Selected.code),

    // 1. Génération de la référence unique
    UpdateContext({
        varSubmissionRef: "SUB-" & Text(Now(), "yyyymmdd-hhmmss")
    });

    // 2. Enregistrement dans MCHF Submissions
    Patch(
        'MCHF Submissions', 
        Defaults('MCHF Submissions'), 
        {
            Title: varSubmissionRef, 
            'Constituent ID': If(tglNewDonor.Value, "NEW", txtConstituentId.Text), 
            'Constituent Name': If(tglNewDonor.Value, "Nouveau donateur", varDonorName),
            'Constituent System ID': If(tglNewDonor.Value, "", varDonorSystemId), 
            'Payment Type': ddPaymentType.Selected.Value,
            'Gift Date': dpGiftDate.SelectedDate, 
            'Gift Type': ddGiftType.Selected.Value,
            'Gift Amount': Value(txtGiftAmount.Text),
            Fees: If(IsBlank(txtFees.Text), Blank(), Value(txtFees.Text)),
            'Net Amount': If(IsBlank(txtFees.Text), Blank(), Value(txtGiftAmount.Text) - Value(txtFees.Text)),
            Campaign: ddCampaign.Selected.code, 
            Appeal: ddAppeal.Selected.code,
            Fund: ddFund.Selected.code,
            Processing status: {Value: "Submitted"},
            'Payment Reference': Coalesce(varPaymentRecord.Title, varPaymentIdParam, "")
        }
    );

    // 3. Liaison en retour dans MCHF - Payment Log
    If(
        !IsBlank(varPaymentRecord),
        Patch(
            'MCHF - Payment Log',
            varPaymentRecord,
            {
                'Submission Reference': varSubmissionRef
            }
        )
    );

    // 4. Modale de confirmation et réinitialisation de la validation
    Notify(
        "Soumission réussie. Référence : " & varSubmissionRef, 
        NotificationType.Success
    );
    UpdateContext({ 
        varShowSuccessPopup: true,
        varTried: false 
    }),

    // Notification si le formulaire est invalide
    Notify("Some fields need your attention.", NotificationType.Error)
)
```

---

### Fenêtre Modale de Confirmation

- **`rectOverlay.Visible`** :
```powerfx
varShowSuccessPopup
```

- **`rectModalCard.Visible`** :
```powerfx
varShowSuccessPopup
```

- **`lblSuccessTitle.Text`** :
```powerfx
"Soumission réussie"
```

- **`lblSuccessTitle.Visible`** :
```powerfx
varShowSuccessPopup
```

- **`lblSuccessRef.Text`** :
```powerfx
"Référence : " & varSubmissionRef
```

- **`lblSuccessRef.Visible`** :
```powerfx
varShowSuccessPopup
```

- **`btnClosePopup.Visible`** :
```powerfx
varShowSuccessPopup
```

- **`btnClosePopup.OnSelect`** :
```powerfx
UpdateContext({ varShowSuccessPopup: false })
```

---

## 4. Tests & Acceptance Criteria Validation
- **Résolution Connexion RENXT (Erreur 401)** : Connexion autorisée avec succès dans l'environnement Établi ; chargement automatique des listes au démarrage sans avertissement[cite: 50].
- **Listes d'imputation dynamiques** : `ddCampaign`, `ddAppeal` et `ddFund` affichent les codes et libellés réels (`AC-2025-2026...`, `EV-GOLF-2026...`, `HA-GENE0001...`).
- **Lookup Donateur** :
  - Saisie de `12345` : Message d'erreur « Enter the 7-digit Constituent ID. »[cite: 52].
  - Saisie de `1234567` : Récupération confirmée du donateur `Camille Exemple` affiché sans superposition[cite: 52].
- **Calcul net** : Montant net calculé dynamiquement à $97.50 (100.00$ de don brut et 2.50$ de frais), le montant brut restant intact à 100.00$.
- **Validation bloquante** : Blocage vérifié lors d'une tentative de soumission sans donateur validé ou sans sélection d'options.
- **Liaison Payment Log** : Rétro-écriture validée de `SUB-YYYYMMDD-HHMMSS` dans la colonne `'Submission Reference'` de `MCHF - Payment Log`.

---

## 5. Technical Notes
- **Gestion des versions dans Établi** : Conformément à la conception de la plateforme Établi (chapitre 8.5 du manuel d'utilisation), chaque enregistrement (`Ctrl + S`) crée directement une version active sans étape distincte de publication web[cite: 45, 49, 83]. Le suivi d'équipe et la revue de code s'effectuent via les Demandes de modification (*Change Requests*) et le dépôt Git associé[cite: 54, 57].