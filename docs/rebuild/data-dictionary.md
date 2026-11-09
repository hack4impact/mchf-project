# Data Dictionary & Storage Structure (MCHF Donation Coding)

**Branch:** `feature/1-data-dictionary`
**Source of Truth:** PRD v0.4, Section 7 (Core information & Business rules)
**Target Environment:** Établi / Power Platform (`Équipe`)
**Solution Package:** `lab/solutions/solution.json` (Solution: `MCHF-Data-Dictionary`)

---

## 1. Schema Specifications

### 1.1 List: `MCHF Submissions`
Stores one record per coded donation submitted by fundraisers.
*Key: `8739ddec-2c50-4b13-b7e1-9a09019faa77`*

| Display Name | Internal Name (Backend / Flows) | Établi / SharePoint Type | Required | Source / Options | Validation & Business Rules |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Title** | `Title` | Single line of text | Yes | System generated | Format: `SUB-00001` (assigned post-creation as `SUB-` + 5-digit ID). Temporary value `"NEW"` on initial save. Max 255 chars. |
| **Constituent ID** | `ConstituentID` | Single line of text | Yes*|User Input|Exactly 7 numeric digits (`\d{7}`).Max 7 chars.*Optional only if "New donor" = Yes. |
| **Constituent Name** | `ConstituentName` | Single line of text | Yes | RE NXT lookup | Populated automatically via constituent lookup. |
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
| **Package** | `Package` | Single line of text | No | RE NXT SKY API | Filtered dynamically based on selected Appeal. |
| **Assigned Solicitor** | `Assigned_x0020_Solicitor` | Single line of text | No | RE NXT SKY API | Credited solicitor. Must never default to the submitting user (`User()`). |
| **Payment Reference** | `Payment_x0020_Reference` | Single line of text | No | Linked Payment | Holds the `PAY-00001` reference when claimed from the Payment Log Register; empty otherwise. Filled by prefilled form (#8). |
| **Processing status** | `Status` | Choice | Yes | Configured choices | Options: `Submitted`, `Under Review`, `Needs Information`, `Rejected`, `Recording in Progress`, `Added to NXT Batch`, `Transfer Failed`, `Cancelled`. Default: `Submitted`. Transition logic owned by Yanis (#4). |
| **Reviewer Email** | `ReviewerEmail` | Single line of text | No | Review app (#15) | Gift Processing member who claimed the submission (#4). |
| **Review Comment** | `ReviewComment` | Multiple lines of text | No | Review app (#15) | Last comment from Return, Reject or Cancel. Shown to the fundraiser. |
| **Approved By Email** | `ApprovedByEmail` | Single line of text | No | Review app (#15) | Who approved. Shown in the audit PDF (#11). |
| **Approved At** | `ApprovedAt` | Date and Time | No | Review app (#15) | When it was approved. Shown in the audit PDF (#11). |
| **Audit PDF** | `AuditPDF` | Single line of text | No | Flow `MCHF - ApproveGift` (#15) | Link to the audit PDF in `MCHF - Audit documents`. Text column: the lab has no hyperlink column. |
| **Audit HTML** | `AuditHTML` | Multiple lines of text | No | Flow `MCHF - ApproveGift` (#15) | Fallback if the lab's file actions are not ready. Not used: the PDF chain works in the lab. |

> *Note on Platform Constraints:* In Établi, the minimum constraint UI for Currency columns accepts integer values only (set to `0`). Strict validation (`Value > 0.00`) is enforced at the Power Fx application layer.

---

### 1.2 List: `MCHF Fund Lines`
Stores individual fund allocations when a donation is split across multiple designations.
*Key: `240294e9-4751-4259-8c04-cd2c60fd746f`*

| Display Name | Internal Name (Backend / Flows) | Établi / SharePoint Type | Required | Description & Rules |
| :--- | :--- | :--- | :--- | :--- |
| **Title** | `Title` | Single line of text | Yes | Fund lookup ID / code from RE NXT (e.g., `HA-CARD0110`). Max 255 chars. |
| **Submission** | `Submission` | Lookup | Yes | Lookup to `MCHF Submissions` (target column: `Title`, target list key: `8739ddec-2c50-4b13-b7e1-9a09019faa77`). |
| **Submission Reference** | `Submission_x0020_Reference` | Single line of text | No | Denormalized `SUB-00001` text identifier for fail-safe referencing. |
| **Amount** | `Amount` | Currency (CAD) | Yes | Allocation amount (Min: $0.01). Enforced rule: `Sum(Fund Lines) == Gift Amount`. |

---

### 1.3 Configurable Option Lists (Managed by Foundation)

Option Lists are maintained in separate lists so administrators can update options without code modifications.

#### `MCHF Payment Types`
*Key: `090bc4f3-574a-4cd4-824b-9c2c93a6bb20`*
* **Schema:** `Title` (Single line of text, required), `Active` (Boolean, default `true`), `Sort_x0020_Order` (Number, required).
* **Initial Seed Data:**
    1. `Cash` (Sort_x0020_Order: 1, Active: true)
    2. `Personal Cheque` (Sort_x0020_Order: 2, Active: true)
    3. `Business Cheque` (Sort_x0020_Order: 3, Active: true)
    4. `Credit Card` (Sort_x0020_Order: 4, Active: true)
    5. `Direct Debit` (Sort_x0020_Order: 5, Active: true)
    6. `EFT` (Sort_x0020_Order: 6, Active: true)
    7. `Stock Donation` (Sort_x0020_Order: 7, Active: true)
    8. `Other` (Sort_x0020_Order: 8, Active: true)

#### `MCHF Gift Types`
*Key: `28d68322-8a1a-424c-8e42-ce784eb6cddd`*
* **Schema:** `Title` (Single line of text, required), `Active` (Boolean, default `true`), `Sort_x0020_Order` (Number, required).
* **Initial Seed Data (In-Scope MVP):**
    1. `One-time gift` (Sort_x0020_Order: 1, Active: true)
    2. `Pledge` (Sort_x0020_Order: 2, Active: true)
    3. `Pledge Payment` (Sort_x0020_Order: 3, Active: true)
    *(Out of base scope: `Gift in Kind`, `Monthly`).*

---

## 2. Reference Formats & Cross-Team Contract

* **Submission Reference:** Formatted as `SUB-00000` (e.g. `SUB-00042`), derived from the auto-increment list ID via `SUB- & Text(ID, "00000")`.
* **Payment Reference:** Formatted as `PAY-00000` (e.g. `PAY-00042`), generated by Dev 3 on the Payment Log Register.
* **Linkage Contract:**
    * In `MCHF Submissions`: Column `Payment_x0020_Reference` stores the linked payment ID when a gift is claimed.
    * In `MCHF Payment Log` (Dev 3): Column `SubmissionReference` stores `varSub.Title` once claimed and coded.

---

## 3. Future Entity Relationships (Out of Base Scope)

The architecture accommodates the following future modules without breaking current core storage:
* **Pledges:** 1:N relationship between a Submission (`Pledge`) and an Installment Schedule list (installment number, due date, installment amount, status).
* **Pledge Payments:** N:1 lookup relationship linking a `Pledge Payment` submission to an existing Pledge ID in NXT (including third-party pledges).
* **Soft Credits:** 1:N relationship from `MCHF Submissions` to a `Soft Credits` child list (linked Constituent ID, Constituent Name, credited amount).
* **Tributes:** 1:1 relationship linking a Submission to Tribute Details (tribute type, tribute constituent, notification card description and delivery date).
* **Events:** N:1 lookup linking the submission to an NXT Event (Event ID, Event Name, ticket/fee classification).
* **Proposals (Opportunities):** N:1 lookup linking major gifts to an NXT Opportunity record.

---

## 4. Open Questions for Marc (Product Manager)

1. **Pledge Schedule Granularity:** PRD v0.4 specifies annual installments for MVP. Will future phases require support for semi-annual or quarterly schedules in the installment sub-structure?
2. **Thank-You Letter Coding:** While receipt amounts and benefit deductions are collected, which automated rule determines the specific letter code for acknowledgements (especially bilingually)?
3. **Pledge Payment Activation:** Should the pledge payment lookup section appear automatically based on `Gift Type == "Pledge Payment"` or via a standalone Yes/No trigger ("Is this a pledge payment?") as suggested in earlier notes?