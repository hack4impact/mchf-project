# Data Dictionary & Storage Structure (MCHF Donation Coding)

**Branch:** `feature/49-sprint4-lists`
**Source of Truth:** PRD v0.4, Section 6, Section 7 & Section 9
**Target Environment:** Établi / Power Platform (`Équipe`)
**Solution Package:** `lab/solutions/solution.json` (Solution: `MCHF-Data-Dictionary`)

---

## 1. Schema Specifications

### 1.1 List: `MCHF Submissions`
Stores one record per coded donation submitted by fundraisers.
*Key: `d92675c6-cec4-453c-8b5a-9dbb7196ba87`*

| Display Name | Internal Name (Backend / Flows) | Établi / SharePoint Type | Required | Source / Options | Validation & Business Rules |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Title** | `Title` | Single line of text | Yes | System generated | Format: `SUB-00001` (assigned post-creation as `SUB-` + 5-digit ID). Temporary value `"NEW"` on initial save. Max 255 chars. |
| **Constituent ID** | `ConstituentID` | Single line of text | No in the list; required by the form unless "New Donor" = Yes | User Input | Exactly 7 numeric digits (`\d{7}`). Max 7 chars. Optional at list level (a list column cannot be required on a condition): the coding form enforces it unless "New donor" = Yes. (PRD acceptance #19/ #25). |
| **Constituent Name** | `ConstituentName` | Single line of text | Yes | RE NXT lookup | Populated automatically via constituent lookup, or entered manually for a new donor. |
| **Constituent System ID** | `Constituent_x0020_System_x0020_I` | Single line of text | No | RE NXT lookup | Numerical SKY API internal system ID (e.g. `11001`). Used by backend flows. |
| **Payment Type** | `Payment_x0020_Type` | Single line of text | Yes | `MCHF Payment Types` | Selected from active options. For pledges, represents the intended future payment method. |
| **Gift Date** | `Gift_x0020_Date` | Date and Time (Date only) | Yes | User Input | Date donation/pledge was obtained. Past dates are explicitly permitted. |
| **Gift Type** | `Gift_x0020_Type` | Single line of text | Yes | `MCHF Gift Types` | Options in scope: `One-time gift`, `Pledge`, `Pledge Payment`. |
| **Gift Amount** | `Gift_x0020_Amount` | Currency (CAD) | Yes | User Input | Transaction gross value. Minimum: $0.01 (enforced by form validation). Never reduced by fees or benefits. |
| **Fees** | `Fees` | Currency (CAD) | No | User Input | Processing / transaction fees incurred. |
| **Net Amount** | `Net_x0020_Amount` | Currency (CAD) | No | Calculated | `Gift Amount - Fees`. Calculated client-side. |
| **Campaign** | `Campaign` | Single line of text | Yes | RE NXT SKY API | Lookup ID/Code from RE NXT (e.g., `AC-2026-2027`). |
| **Appeal** | `Appeal` | Single line of text | Yes | RE NXT SKY API | Lookup ID/Code from RE NXT (e.g., `EV-GOLF-2026`). Governs Package options. |
| **Fund** | `Fund` | Single line of text | Yes | RE NXT SKY API | Primary fund lookup ID/Code (e.g., `HA-CARD0110`). |
| **Package** | `Package` | Single line of text | No | RE NXT SKY API | Filtered dynamically based on selected Appeal (free text in lab). |
| **Assigned Solicitor** | `Assigned_x0020_Solicitor` | Single line of text | No | RE NXT SKY API | Credited solicitor 7-digit Constituent ID. Must never default to the submitting user (`User()`). |
| **Payment Reference** | `Payment_x0020_Reference` | Single line of text | No | Linked Payment | Holds the `PAY-00001` reference when claimed from the Payment Log Register; empty otherwise. Filled by prefilled form (#8). |
| **Processing status** | `Status` | Choice | Yes | Configured choices | Options: `Submitted`, `Under Review`, `Needs Information`, `Rejected`, `Recording in Progress`, `Added to NXT Batch`, `Transfer Failed`, `Cancelled`. Default: `Submitted`. Transition logic owned by Yanis (#4). |
| **Reviewer Email** | `ReviewerEmail` | Single line of text | No | Review app (#15) | Gift Processing member who claimed the submission (#4). |
| **Review Comment** | `ReviewComment` | Multiple lines of text | No | Review app (#15) | Last comment from Return, Reject or Cancel. Shown to the fundraiser. |
| **Approved By Email** | `ApprovedByEmail` | Single line of text | No | Review app (#15) | Who approved. Shown in the audit PDF (#11). |
| **Approved At** | `ApprovedAt` | Date and Time | No | Review app (#15) | When it was approved. Shown in the audit PDF (#11). |
| **Audit PDF** | `AuditPDF` | Single line of text | No | Flow `MCHF - ApproveGift` (#15) | Link to the audit PDF in `MCHF - Audit documents`. Text column: the lab has no hyperlink column. |
| **Audit HTML** | `AuditHTML` | Multiple lines of text | No | Flow `MCHF - ApproveGift` (#15) | Fallback if the lab's file actions are not ready. Not used: the PDF chain works in the lab. |
| **Notes** | `Notes` | Multiple lines of text | No | User Input / Payment Log | Free-text administrative or payment notes (pre-filled from `MCHF - Payment Log` when applicable). |
| **Tax Receipt** | `TaxReceipt` | Yes/No | No | User Input | Default: `No` (`false`). Indicates whether an official tax receipt is to be issued. |
| **Receipt Amount** | `ReceiptAmount` | Currency (CAD) | No | User Input / Calculated | Default: `Gift Amount - Benefit Amount`, between `0` and `Gift Amount`, empty when `Tax Receipt = No`. |
| **Benefit Amount** | `BenefitAmount` | Currency (CAD) | No | User Input | Value of any advantage/benefit received by the donor (CAD, minimum `0`, at most `Gift Amount`). |
| **New Donor** | `NewDonor` | Yes/No | No | User Input | Default: `No` (`false`). Indicates the submission is for a donor not yet registered in RE NXT. |
| **Soft Credit** | `SoftCredit` | Yes/No | No | User Input | Default: `No` (`false`). Answer to `"Is there a soft credit to apply?"` (#54). |
| **Pledge Payment** | `PledgePayment` | Yes/No | No | User Input | Default: `No` (`false`). Answer to `"Is this a pledge payment?"` (#53). |
| **Installment Count** | `InstallmentCount` | Number (0 decimals) | No | User Input | Minimum `1`, maximum `5`. Empty when `Gift Type <> "Pledge"` (#55). |
| **Installment Frequency** | `InstallmentFrequency` | Single line of text | No | User Input / Fixed | `"Annual"` for pledges; empty when `Gift Type <> "Pledge"` (#55). |
| **First Installment Date** | `FirstInstallmentDate` | Date and Time (Date only) | No | User Input | Scheduled date of the first installment; empty when `Gift Type <> "Pledge"` (#55). |

> *Note on Platform Constraints:* As of Établi release `2026-11-08.2`, Currency columns support decimal minimum/maximum constraints (e.g., `0.01` for `Gift Amount`, while `Receipt Amount` and `Benefit Amount` use `0` per Sprint 3 specifications). Strict validation is also enforced at the Power Fx application layer.

---

### 1.2 List: `MCHF Fund Lines`
Stores individual fund allocations when a donation is split across multiple designations.
*Key: `3003a026-5bf8-44d3-a426-b572609011a1`*

| Display Name | Internal Name (Backend / Flows) | Établi / SharePoint Type | Required | Description & Rules |
| :--- | :--- | :--- | :--- | :--- |
| **Title** | `Title` | Single line of text | Yes | Fund lookup ID / code from RE NXT (e.g., `HA-CARD0110`). Max 255 chars. |
| **Submission** | `Submission` | Lookup | Yes | Lookup to `MCHF Submissions` (target column: `Title`, target list key: `d92675c6-cec4-453c-8b5a-9dbb7196ba87`). |
| **Submission Reference** | `Submission_x0020_Reference` | Single line of text | No | Denormalized `SUB-00001` text identifier for fail-safe referencing. |
| **Amount** | `Amount` | Currency (CAD) | Yes | Allocation amount (Min: $0.01). Enforced rule: `Sum(Fund Lines) == Gift Amount`. |

---

### 1.3 Configurable Option Lists (Managed by Foundation)

Option Lists are maintained in separate lists so administrators can update options without code modifications.

#### `MCHF Payment Types`
*Key: `14db9871-d248-4373-9953-ad47534c3e95`*
- **Schema:** `Title` (Single line of text, required), `Active` (Boolean, default `true`), `Sort_x0020_Order` (Number, required).
- **Initial Seed Data:**
  1. `Cash` (Sort_x0020_Order: 1, Active: true)
  2. `Personal Cheque` (Sort_x0020_Order: 2, Active: true)
  3. `Business Cheque` (Sort_x0020_Order: 3, Active: true)
  4. `Credit Card` (Sort_x0020_Order: 4, Active: true)
  5. `Direct Debit` (Sort_x0020_Order: 5, Active: true)
  6. `EFT` (Sort_x0020_Order: 6, Active: true)
  7. `Stock Donation` (Sort_x0020_Order: 7, Active: true)
  8. `Other` (Sort_x0020_Order: 8, Active: true)

#### `MCHF Gift Types`
*Key: `c842f564-f45a-4b06-85c3-d92e02810f69`*
- **Schema:** `Title` (Single line of text, required), `Active` (Boolean, default `true`), `Sort_x0020_Order` (Number, required).
- **Initial Seed Data (In-Scope MVP):**
  1. `One-time gift` (Sort_x0020_Order: 1, Active: true)
  2. `Pledge` (Sort_x0020_Order: 2, Active: true)
  3. `Pledge Payment` (Sort_x0020_Order: 3, Active: true)
    *(Out of base scope: `Gift in Kind`, `Monthly`).*

---

### 1.4 List: `MCHF - Soft Credits`
Stores one child line per constituent credited with a soft credit on a submission (`SoftCredit = Yes`, #54).
*Key: `cd0090e1-5197-4ba7-9675-f406273e9d0c`*

| Display Name | Internal Name (Backend / Flows) | Établi / SharePoint Type | Required | Description & Rules |
| :--- | :--- | :--- | :--- | :--- |
| **Title** | `Title` | Single line of text | Yes | Credited constituent 7-digit `lookup_id` (mirrors `ConstituentID`). Max 255 chars. |
| **Submission** | `Submission` | Lookup | No (enforced by form) | Lookup to `MCHF Submissions` (target column: `Title`, target list key: `d92675c6-cec4-453c-8b5a-9dbb7196ba87`). |
| **Submission Reference** | `Submission_x0020_Reference` | Single line of text | No | Denormalized `SUB-00001` text identifier for fail-safe referencing and OData filtering. |
| **Constituent ID** | `ConstituentID` | Single line of text | No (enforced by form) | Credited constituent 7-digit `lookup_id`. Max 7 chars. |
| **Constituent Name** | `ConstituentName` | Single line of text | No (enforced by form) | Read-only constituent name returned by RE NXT lookup. |
| **Amount** | `Amount` | Currency (CAD) | No (enforced by form) | Soft credit amount (Min: $0.01). No total cap enforced against `Gift Amount` in the lab. |

---

### 1.5 List: `MCHF - Pledge Payments`
Stores one child line per open NXT pledge paid by a pledge payment submission (`PledgePayment = Yes`, #53).
*Key: `7d01bbe7-48c2-41ad-9042-5063b2a38483`*

| Display Name | Internal Name (Backend / Flows) | Établi / SharePoint Type | Required | Description & Rules |
| :--- | :--- | :--- | :--- | :--- |
| **Title** | `Title` | Single line of text | Yes | Pledge system ID in RE NXT (mirrors `PledgeID`). Max 255 chars. |
| **Submission** | `Submission` | Lookup | No (enforced by form) | Lookup to `MCHF Submissions` (target column: `Title`, target list key: `d92675c6-cec4-453c-8b5a-9dbb7196ba87`). |
| **Submission Reference** | `Submission_x0020_Reference` | Single line of text | No | Denormalized `SUB-00001` text identifier for fail-safe referencing and OData filtering. |
| **Pledge ID** | `PledgeID` | Single line of text | No (enforced by form) | System ID of the open pledge in RE NXT. |
| **Pledge Constituent ID** | `PledgeConstituentID` | Single line of text | No (enforced by form) | 7-digit Constituent ID of the pledge owner (donor or third party). Max 7 chars. |
| **Pledge Constituent Name** | `PledgeConstituentName` | Single line of text | No (enforced by form) | Display name of the pledge owner returned by RE NXT. |
| **Pledge Balance** | `PledgeBalance` | Currency (CAD) | No (enforced by form) | Open pledge balance (`balance.value`) observed at coding time. |
| **Amount** | `Amount` | Currency (CAD) | No (enforced by form) | Payment amount applied to this pledge (Min: $0.01, at most `PledgeBalance`). Enforced rule: `Sum(Pledge Payments) == Gift Amount`. |

---

### 1.6 List: `MCHF - Pledge Installments`
Stores the annual installment schedule lines generated for a pledge submission (`Gift Type = "Pledge"`, #55).
*Key: `3d207ee1-2b06-4944-9f98-06d8c3b3a513`*

| Display Name | Internal Name (Backend / Flows) | Établi / SharePoint Type | Required | Description & Rules |
| :--- | :--- | :--- | :--- | :--- |
| **Title** | `Title` | Single line of text | Yes | Sequence number of the installment as text (`"1"`, `"2"`, …). Max 255 chars. |
| **Submission** | `Submission` | Lookup | No (enforced by form) | Lookup to `MCHF Submissions` (target column: `Title`, target list key: `d92675c6-cec4-453c-8b5a-9dbb7196ba87`). |
| **Submission Reference** | `Submission_x0020_Reference` | Single line of text | No | Denormalized `SUB-00001` text identifier for fail-safe referencing and OData filtering. |
| **Installment Number** | `InstallmentNumber` | Number (0 decimals) | No (enforced by form) | Installment sequence integer (`1` to `5`). |
| **Due Date** | `DueDate` | Date and Time (Date only) | No (enforced by form) | Scheduled annual due date (`FirstInstallmentDate + (InstallmentNumber - 1) years`). |
| **Amount** | `Amount` | Currency (CAD) | No (enforced by form) | Installment amount (Min: $0.01). Enforced rule: `Sum(Pledge Installments) == Gift Amount`. |

---

## 2. Reference Formats & Cross-Team Contract

- **Submission Reference:** Formatted as `SUB-00000` (e.g. `SUB-00042`), derived from the auto-increment list ID via `SUB- & Text(ID, "00000")`.
- **Payment Reference:** Formatted as `PAY-00000` (e.g. `PAY-00042`), generated by Dev 3 on the Payment Log Register.
- **Linkage Contract:**
  - In `MCHF Submissions`: Column `Payment_x0020_Reference` stores the linked payment ID when a gift is claimed.
  - In `MCHF Payment Log` (Dev 3): Column `SubmissionReference` stores `varSub.Title` once claimed and coded.

---

## 3. Future Entity Relationships (Sprint 5+ Scope)

The architecture accommodates the following remaining Sprint 5+ modules without breaking current core storage:
- **Tributes:** 1:1 relationship linking a Submission to Tribute Details (tribute type, tribute constituent, notification card description and delivery date).
- **Events:** N:1 lookup linking the submission to an NXT Event (Event ID, Event Name, ticket/fee classification).
- **Proposals (Opportunities):** N:1 lookup linking major gifts to an NXT Opportunity record.

---

## 4. Open Questions for Marc (Product Manager)

1. **Pledge Schedule Granularity:** PRD v0.4 specifies annual installments for MVP. Will future phases require support for semi-annual or quarterly schedules in the installment sub-structure?
2. **Thank-You Letter Coding:** While receipt amounts and benefit deductions are collected, which automated rule determines the specific letter code for acknowledgements (especially bilingually)?
3. **Pledge Payment Activation:** Should the pledge payment lookup section appear automatically based on `Gift Type == "Pledge Payment"` or via a standalone Yes/No trigger ("Is this a pledge payment?") as suggested in earlier notes?