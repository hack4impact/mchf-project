# Rebuild Guide: Coding Form (Simple Gift - Issue #8)

## 1. Screens & Layout
- **Écran principal** : `Screen1` (Format Tablette 1366 × 768).

### Entête
- `rectHeaderBg` (Rectangle) : Bandeau supérieur de couleur vert foncé.
- `lblHeader` (Label) : Titre de l'application (« MCHF - Coding Form »).

### Colonne de gauche (Identification Donateur & Montants)
- **Donateur** :
  - `lblDonatorID` (Label) : Libellé « ID Donateur ».
  - `txtConstituentId` (Text Input) : Champ de saisie pour l'identifiant donateur (7 chiffres).
  - `btnSearchDonor` (Button) : Bouton de déclenchement « Lookup » (recherche via API RENXT).
  - `lblLookup` (Label) : Affichage du statut ou du résultat de la recherche (ex. « Aucun donateur sélectionné »).
  - `tglNewDonor` (Toggle) : Bascule « Nouveau donateur ? ».
  - `lblNewDonator` (Label) : Texte descriptif associé à la bascule.
  - `btnNewDonorLink` (Button / Link) : Action d'aide ou de création de fiche donateur.
- **Montants & Calculs** :
  - `lblGiftAmount` (Label) : Libellé « Montant du don ($) ».
  - `txtGiftAmount` (Text Input) : Saisie du montant brut du don.
  - `lblFees` (Label) : Libellé « Frais ($) ».
  - `txtFees` (Text Input) : Saisie des frais de traitement bancaire.
  - `lblNetAmountText` (Label) : Libellé statique « Montant net : ».
  - `lblNetAmount` (Label) : Affichage dynamique du montant net calculé.

### Colonne de droite (Détails & Imputation comptable)
- `lblGiftDate` (Label) : Libellé « Date du don ».
- `dpGiftDate` (Date Picker) : Sélection de la date de réception du don (autorise les dates passées).
- `lblGiftType` (Label) : Libellé « Type de don ».
- `ddGiftType` (Dropdown) : Menu déroulant pour le type de don (`One-time gift`, `Pledge`, `Pledge Payment`).
- `lblPaymentType` (Label) : Libellé « Mode de paiement ».
- `ddPaymentType` (Dropdown) : Menu déroulant du mode de règlement (`Credit Card`, `Cash`, `Cheque`, etc.).
- `lblCampaign` (Label) : Libellé « Campagne ».
- `ddCampaign` (Dropdown) : Menu déroulant d'attribution de la campagne.
- `lblAppeal` (Label) : Libellé « Appel ».
- `ddAppeal` (Dropdown) : Menu déroulant d'affectation de l'appel.
- `lblFunds` (Label) : Libellé « Fonds ».
- `ddFund` (Dropdown) : Menu déroulant d'imputation du fonds.
- `btnSubmit` (Button) : Bouton d'enregistrement et de soumission finale (« Submit »).

### Fenêtre modale de confirmation (Succès)
- `rectOverlay` (Rectangle) : Fond grisé semi-transparent couvrant l'écran (`Visible: varShowSuccessPopup`).
- `rectModalCard` (Rectangle) : Carte centrale blanche du pop-up (`Visible: varShowSuccessPopup`).
- `lblSuccessTitle` (Label) : Titre de confirmation (« Don enregistré ») (`Visible: varShowSuccessPopup`).
- `lblSuccessRef` (Label) : Affichage de la référence unique générée (`varSubmissionRef`) (`Visible: varShowSuccessPopup`).
- `btnClosePopup` (Button) : Bouton pour fermer la fenêtre modale (`Visible: varShowSuccessPopup`).

---

## 2. Business Rules & Logic
- **Calcul dynamique du montant net** :
  - Formule : `Value(txtGiftAmount.Text) - Value(txtFees.Text)`.
  - Règle stricte d'acceptation : Le montant brut saisi reste intact dans `txtGiftAmount`. Un don de 100.00$ avec 2.50$ de frais affiche un net de 97.50$ sans modifier les 100.00$.
- **Gestion du donateur** :
  - Si `tglNewDonor.Value` est activé, l'enregistrement force `Constituent ID` à `"NEW"` et `Constituent Name` à `"Nouveau donateur"`.
  - Si désactivé, l'enregistrement utilise l'ID saisi et le nom issu du lookup (`varDonor.name`).
- **Génération de la référence unique** :
  - Clé générée au format `SUB-YYYYMMDD-HHMMSS` via `Text(Now(), "yyyymmdd-hhmmss")`.
- **Intégrité de statut (SharePoint)** :
  - Le champ `Processing status` requiert la valeur initiale obligatoire `{ Value: "Submitted" }`.
- **Liaison Journal de paiement (Payment Log)** :
  - Lecture du paramètre d'URL `paymentId` via `Param("paymentId")`.
  - Si un enregistrement correspondant existe dans la liste `MCHF - Payment Log`, son statut passe à `{ Value: "Coded" }` dès la soumission.

---

## 3. PowerFX Formulas

### `Screen1.OnVisible`
```powerfx
// Initialisation du paramètre d'URL et préchargement du paiement associé
Set(varPaymentIdParam, Param("paymentId"));
Set(
    varPaymentRecord, 
    If(
        !IsBlank(varPaymentIdParam), 
        LookUp('MCHF - Payment Log', ID = Value(varPaymentIdParam) || Title = varPaymentIdParam), 
        Blank()
    )
);

// Initialisation des variables de session et réinitialisation des statuts
Set(varDonor, Blank());
Set(varDonorStatus, "");
Set(varSubmissionRef, "");
Set(varShowSuccessPopup, false);
```

### Menus déroulants (Items)
```powerfx
// ddCampaign.Items
["Campagne Annuelle 2026", "Fonds d'urgence pédiatrique"]

// ddAppeal.Items
["Appel Web / En ligne", "Publipostage Automne"]

// ddFund.Items
["Fonds général non restreint", "Recherche en cardiologie"]
```

### `lblNetAmount.Text`
```powerfx
Text(
    Value(txtGiftAmount.Text) - Value(txtFees.Text),
    "$#,##0.00"
)
```

### `btnSubmit.OnSelect`
```powerfx
// 1. Génération de la référence unique
UpdateContext({
    varSubmissionRef: "SUB-" & Text(Now(), "yyyymmdd-hhmmss")
});

// 2. Enregistrement principal dans MCHF Submissions
Patch(
    'MCHF Submissions',
    Defaults('MCHF Submissions'),
    {
        Title: varSubmissionRef,
        'Processing status': { Value: "Submitted" },
        'Constituent ID': If(tglNewDonor.Value, "NEW", txtConstituentId.Text),
        'Constituent Name': If(tglNewDonor.Value, "Nouveau donateur", Coalesce(varDonor.name, txtConstituentId.Text)),
        'Payment Type': ddPaymentType.Selected.Titre,
        'Gift Date': dpGiftDate.SelectedDate,
        'Gift Type': ddGiftType.Selected.Titre,
        'Gift Amount': Value(txtGiftAmount.Text),
        Fees: Value(txtFees.Text),
        'Net Amount': Value(txtGiftAmount.Text) - Value(txtFees.Text),
        Campaign: ddCampaign.Selected.Value,
        Appeal: ddAppeal.Selected.Value,
        Fund: ddFund.Selected.Value
    }
);

// 3. Mise à jour de liaison dans MCHF - Payment Log si applicable
If(
    !IsBlank(varPaymentRecord),
    Patch(
        'MCHF - Payment Log',
        varPaymentRecord,
        {
            Status: { Value: "Coded" }
        }
    )
);

// 4. Notification Toast et ouverture du pop-up de confirmation
Notify(
    "La soumission " & varSubmissionRef & " a été enregistrée avec succès.",
    NotificationType.Success
);
UpdateContext({ varShowSuccessPopup: true });
```

### `btnClosePopup.OnSelect`
```powerfx
UpdateContext({ varShowSuccessPopup: false });
```

---

## 4. Tests & Acceptance Criteria Validation
- **Calcul net (100$ don, 2.50$ frais)** : Montant net calculé à 97.50$, montant brut maintenu à 100.00$.
- **Dates passées** : Prise en compte de dates antérieures sans message d'erreur.
- **Enregistrement SharePoint** : Entrée insérée avec succès dans `MCHF Submissions` avec le statut `{ Value: "Submitted" }`.
- **Modale de succès** : Affichage validé de l'identifiant `SUB-...` dans la pop-up `rectModalCard`.

---

## 5. Known Limitations & Technical Notes
- **Connecteur API RENXT (Erreur HTTP 401)** : L'authentification OAuth et le tunnel Cloudflare de l'instance simulée étaient indisponibles lors des tests ; les données d'options (`Campaign`, `Appeal`, `Fund`) ont été intégrées via des listes statiques conformes au dictionnaire de données (`solution.json`).
- **Export binaire (.msapp)** : L'environnement web conteneurisé du lab Établi ne propose pas d'option d'export de package binaire ; l'ensemble des définitions de structures et codes PowerFX ci-dessus constitue le guide officiel pour la reconstruction de l'application au Sprint 4.