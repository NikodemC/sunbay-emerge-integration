# ERP → Sunbay Collection Data Ingestion - Integration Guide for Emerge

This document specifies the **data Sunbay needs to receive** from an on-premise ERP integration agent, and **how that data should be delivered**, so that Sunbay can run debt-collection (windykacja) on behalf of the end-client.

It is written for the **integration partner (Emerge)** building the agent that reads data from the client's ERP (Comarch, Enova, Symfonia, …) and sends it to Sunbay.

The **data model (§3) is the core of this document.** The delivery mechanics (§4-§9) describe how that data reaches us. A number of points are deliberately left **open for discussion (§10)** - the final shape of some of the protocol depends on how the agent is implemented.

> **Scope.** This document covers *what data we need* and *in what form you send it*. How Sunbay stores, processes, or acts on the data internally is intentionally out of scope.

---

## Contents

1. [Purpose & Scope](#1-purpose--scope)
2. [How Data Reaches Sunbay](#2-how-data-reaches-sunbay)
3. [Data Model](#3-data-model)
4. [Delivery Protocol & Endpoints](#4-delivery-protocol--endpoints)
5. [Sync Modes](#5-sync-modes)
6. [Data Formats & Conventions](#6-data-formats--conventions)
7. [Security & Authentication](#7-security--authentication)
8. [Reliability](#8-reliability)
9. [Scheduling & Volume](#9-scheduling--volume)
10. [Open Questions for Emerge](#10-open-questions-for-emerge)

---

## 1. Purpose & Scope

Sunbay automates collection of overdue receivables (email/SMS reminders and related flows). To do this for an end-client, Sunbay needs a continuous feed of that client's **sales/revenue invoices** (paid and unpaid) together with the **debtor (customer) data** for each invoice, and the **PDF** of each invoice.

Because the integration agent is **installed on the client's own machines**, the connectivity is **one-way**:

- The agent can **send data out** to Sunbay over the public internet.
- Sunbay **cannot initiate connections back** into the client's network or the agent.

Therefore the agent **pushes** data to Sunbay's ingestion endpoints on a schedule. Sunbay is the receiver; the ERP remains the system of record. Sunbay never writes back to the ERP.

**In scope for this document**

- The exact data fields Sunbay needs per invoice and per customer (§3).
- How the data + PDF are delivered (§4), in which formats (§6), how often (§9), and how authenticated (§7).
- Which invoices to send and how to keep them up to date (§5).

**Out of scope**

- What Sunbay does with the data internally and how collection flows are configured.
- Storage of the PDF and any internal references/URLs Sunbay generates for it.

---

## 2. How Data Reaches Sunbay

There are two delivery patterns. The **per-invoice push is the primary one.**

### 2.1 Per-invoice push (primary)

For each invoice, the agent makes **one call** to Sunbay carrying:

1. the invoice's **structured data** (the fields in §3), **including the debtor/customer data** for that invoice, and
2. the **PDF** of that invoice.

Invoices are sent **one after another** on the agent's interval. Each call is small, so a standard HTTPS request is sufficient - no large-file handling is required.

> Data and PDF travel **together** in the same call (recommended: `multipart/form-data`). A two-step variant (data first, then PDF) is possible if it suits the agent better - see §4 and §10.

### 2.2 Bulk data-only (secondary)

For **initial historical backfill**, or for clients where PDFs are not available, invoice + customer **data only** (no PDFs) may be sent as a single **compressed file** (`.zip` of CSV or JSON Lines). The same logical fields from §3 apply, one record per invoice.

This is an alternative to issuing thousands of individual calls for a first load; PDFs for the relevant (typically still-open) invoices can then follow via the per-invoice path.

### 2.3 Acknowledgment

Sunbay accepts each push and processes it. **How Sunbay reports back the per-invoice outcome (accepted / rejected / duplicate) - and whether the agent polls for it - is a point to agree during onboarding** (§4, §10). At minimum, Sunbay returns an HTTP status indicating that a push was received and is well-formed.

---

## 3. Data Model

This is the **most important section**. Each invoice you send to Sunbay should carry the following fields. Customer (debtor) data is **embedded with each invoice** (denormalized) - there is no separate customer feed in the per-invoice path.

**Required** column legend: **Yes** = mandatory · **Rec.** = recommended (strongly preferred) · **Opt.** = optional · **Cond.** = conditional.

### 3.1 Invoice identification

| Field | Type | Required | Description |
|---|---|---|---|
| `externalInvoiceId` | string | **Yes** | Stable, globally-unique identifier of the invoice **in the source ERP**. Used to recognise the same invoice across syncs (deduplication / update key). Must be **stable** - the same invoice must always carry the same id, even after edits. |
| `invoiceNumber` | string | **Yes** | Human-readable invoice number (e.g. `FV/2026/01/0123`). Shown to the debtor in reminders. |
| `documentType` | enum | **Yes** | Kind of document - see §3.1.1. |
| `correctedExternalInvoiceId` | string | **Cond.** | When `documentType = CorrectiveInvoice`: the `externalInvoiceId` of the original invoice this document corrects. |
| `issueDate` | date | **Yes** | Date the invoice was issued. |
| `dueDate` | date | **Yes** | Payment due date - when the invoice becomes collectible. |
| `paymentTermDays` | integer | **Opt.** | Payment term in days, if available. |

#### 3.1.1 `documentType` values

| Value | Meaning (PL) | Collectible receivable? |
|---|---|---|
| `Invoice` | Faktura VAT | **Yes** |
| `CorrectiveInvoice` | Faktura korygująca | Adjusts an existing receivable |
| `AdvanceInvoice` | Faktura zaliczkowa | Yes (advance) |
| `FinalInvoice` | Faktura końcowa | Yes |
| `Proforma` | Faktura proforma | **No** - not a legal receivable |
| `CreditNote` | Nota uznaniowa | Reduces a receivable |
| `DebitNote` | Nota obciążeniowa | Increases a receivable |

> If your ERP uses other document kinds, list them during onboarding so we can map them (§10).

### 3.2 Amounts & currency

| Field | Type | Required | Description |
|---|---|---|---|
| `currency` | string (ISO 4217) | **Yes** | e.g. `PLN`, `EUR`. |
| `amountNet` | decimal | **Yes** | Net amount. |
| `amountVat` | decimal | **Yes** | VAT amount. |
| `amountGross` | decimal | **Yes** | Gross total - the full amount of the receivable. |
| `amountPaid` | decimal | **Yes** | Amount already paid against this invoice (`0` if none). Enables partial-payment handling. |
| `amountOutstanding` | decimal | **Yes** | Remaining balance still owed (`amountGross - amountPaid`). This is what collection chases. |

### 3.3 Status & lifecycle

| Field | Type | Required | Description |
|---|---|---|---|
| `status` | enum | **Yes** | `Open` · `PartiallyPaid` · `Paid` · `Cancelled`. |
| `paidDate` | date | **Cond.** | Date the invoice was fully paid. Required when `status = Paid`. |
| `isBlockedForCollection` | boolean | **Yes** | `true` if Sunbay must **not** chase this invoice (dispute, legal hold, internal block). |

### 3.4 Debtor (customer)

Embedded per invoice. This is the party Sunbay contacts.

| Field | Type | Required | Description |
|---|---|---|---|
| `externalCustomerId` | string | **Yes** | Stable, unique identifier of the customer in the source ERP. |
| `customerName` | string | **Yes** | Debtor name (company or person). |
| `customerTaxId` | string | **Rec.** | Tax identifier (NIP). |
| `customerEmail` | string | **Rec.** | Primary email. Required for email reminders. |
| `customerEmailCcs` | string[] | **Opt.** | **List** of additional CC email addresses. |
| `customerPhone` | string | **Opt.** | Phone number in **international format including the country code**, e.g. `+48512345678`. Required for SMS reminders. |
| `customerAddress` | string | **Rec.** | Postal address. |
| `customerCountryCode` | string (ISO 3166-1) | **Yes** | e.g. `PL`. |
| `customFields` | object | **Opt.** | Arbitrary key-value pairs at **customer** level (see §3.6). |

### 3.5 Seller & payment

| Field | Type | Required | Description |
|---|---|---|---|
| `sellerName` | string | **Opt.** | Issuing entity name. A single ERP installation may contain several legal entities/sellers. |
| `sellerTaxId` | string | **Opt.** | Seller tax identifier (NIP). |
| `bankAccount` | string | **Yes** | Bank account the debtor should pay into (IBAN/NRB). Included in reminders. |

### 3.6 Custom fields & references

| Field | Type | Required | Description |
|---|---|---|---|
| `customFields` | object | **Opt.** | Arbitrary key-value pairs at **invoice** level. **Custom fields are supported at both invoice and customer level** - send any extra attributes that may be useful (segment, region, contract code, cost centre, …). |
| `externalReference` | string | **Opt.** | Any additional reference useful for reconciliation. |

### 3.7 PDF

A **PDF document is required for every invoice**, delivered together with that invoice's data in the same call (§4). One PDF per invoice. PDF specifics (always available? maximum size? PDFs for corrective documents?) are confirmed during onboarding (§10).

### 3.8 Optional / future

Invoice **line items** (per-line name, quantity, net, VAT, gross) are **not requested at this stage** - the PDF already carries them. They may be added later if needed for analytics.

### 3.9 Example payload (per-invoice)

The structured part (the `metadata`), with the PDF delivered alongside it (see §4):

```json
{
  "tenantCode": "ACME-PL",
  "syncMode": "Incremental",
  "invoice": {
    "externalInvoiceId": "COMARCH-2026-FV-000123",
    "invoiceNumber": "FV/2026/01/0123",
    "documentType": "Invoice",
    "correctedExternalInvoiceId": null,
    "issueDate": "2026-01-10",
    "dueDate": "2026-01-24",
    "paymentTermDays": 14,

    "currency": "PLN",
    "amountNet": 1000.00,
    "amountVat": 230.00,
    "amountGross": 1230.00,
    "amountPaid": 0.00,
    "amountOutstanding": 1230.00,

    "status": "Open",
    "paidDate": null,
    "isBlockedForCollection": false,

    "seller": {
      "sellerName": "ACME Sp. z o.o.",
      "sellerTaxId": "5213001234",
      "bankAccount": "PL61109010140000071219812874"
    },

    "customer": {
      "externalCustomerId": "COMARCH-CUST-10001",
      "customerName": "Kowalski Handel Sp. z o.o.",
      "customerTaxId": "7010001234",
      "customerEmail": "ksiegowosc@kowalski.pl",
      "customerEmailCcs": ["zarzad@kowalski.pl", "biuro@kowalski.pl"],
      "customerPhone": "+48512345678",
      "customerAddress": "ul. Przykładowa 12, 00-001 Warszawa",
      "customerCountryCode": "PL",
      "customFields": {
        "segment": "B2B",
        "region": "Mazowieckie"
      }
    },

    "externalReference": "SAP-DOC-998877",
    "customFields": {
      "costCenter": "CC-204",
      "salesRep": "A. Nowak"
    }
  }
}
```

---

## 4. Delivery Protocol & Endpoints

> The endpoints below are a **proposal** to illustrate the intended shape. The **exact paths, request/response bodies, acknowledgment mechanism, and retry behaviour are to be finalised together with Emerge** (see §10) - they depend on how the agent is built and on whether Emerge wants to read back per-invoice results.

### 4.1 Per-invoice push (primary)

`POST /api/ingest/invoices` - one invoice per call, as `multipart/form-data`:

```
POST /api/ingest/invoices
Authorization: <see §7>
Content-Type: multipart/form-data; boundary=----sunbay

------sunbay
Content-Disposition: form-data; name="metadata"
Content-Type: application/json

{ ...the JSON from §3.9... }
------sunbay
Content-Disposition: form-data; name="pdf"; filename="FV-2026-01-0123.pdf"
Content-Type: application/pdf

%PDF-1.7 ...binary...
------sunbay--
```

Sunbay responds with an HTTP status indicating receipt (e.g. `2xx` accepted, `4xx` for a malformed/invalid request). The precise success/error body is part of the acknowledgment design (§4.4).

### 4.2 Bulk data-only (secondary)

`POST /api/ingest/bulk` - a single compressed file (`.zip` of CSV or JSON Lines), **data only, no PDFs** - for backfill or PDF-less clients (§2.2).

### 4.3 Optional "sync session" framing

To bound a full snapshot (§5) and give a natural place to report results, calls in one cycle may be grouped in a session: a **begin** call returns a `sessionId`, per-invoice calls reference it, and a **complete** call closes the cycle. This is optional and subject to the same "to be agreed" note above.

### 4.4 Acknowledgment, errors & duplicates - **to be agreed**

Because Sunbay cannot call the agent, any per-invoice or per-batch result the agent needs (accepted / rejected with reasons / duplicate) must be **read back by the agent**. Whether the agent wants this at all, what response bodies Sunbay returns, how validation errors and duplicates are signalled, and how this ties into retries (§8) - **all of this is a discussion point for onboarding**, not fixed here.

---

## 5. Sync Modes

Which invoices to send each cycle depends on the end-client; both modes below are supported and chosen per client at onboarding.

### 5.1 Full snapshot

Each cycle, the agent sends **all currently-open invoices** (plus recently-paid ones, so settlements are reflected). Simple and self-healing - corrections, cancellations and payments are naturally picked up because the complete current picture is resent.

### 5.2 Incremental

Each cycle, the agent sends **only invoices created or changed since the previous successful sync** (including those whose status changed to paid). Lighter, but **deletions/cancellations in the ERP will not appear as a "change"** - so an **explicit cancellation signal** is required (a `Cancelled` status, or a dedicated cancel call), otherwise Sunbay would keep chasing a debt that no longer exists.

### 5.3 Corrections (korekta)

A corrective document is sent with `documentType = CorrectiveInvoice` and `correctedExternalInvoiceId` pointing at the original. It adjusts the outstanding amount of the original receivable. **How your ERP expresses correction amounts** (the new corrected totals, the difference/delta, or "before/after") must be confirmed so we interpret the balance correctly (§10).

---

## 6. Data Formats & Conventions

| Aspect | Convention |
|---|---|
| **Text encoding** | **UTF-8** for all text and for any CSV/JSON bulk content. ⚠️ Polish ERPs often default to **Windows-1250 / CP1250** - the agent must convert to UTF-8. |
| **Dates** | ISO-8601 calendar dates: `YYYY-MM-DD` (e.g. `2026-01-24`). |
| **Timestamps** | ISO-8601 in **UTC** with `Z` (e.g. `2026-01-24T09:30:00Z`). |
| **Decimal separator** | **Dot** (`.`). No thousands separators. ⚠️ Polish locale uses a comma - the agent must normalise (`1230,00` → `1230.00`). |
| **Currency** | ISO 4217 three-letter code. |
| **Phone** | International format with country code, e.g. `+48512345678`. |
| **Booleans** | `true` / `false`. |
| **Missing values** | Omit the field or send `null` - do not send empty placeholder strings for numeric/date fields. |

---

## 7. Security & Authentication

### 7.1 Transport

- All calls to Sunbay use **HTTPS / TLS 1.2+**.

### 7.2 Authentication (options - chosen per onboarding)

The final method is agreed during onboarding and aligned with the client's security policy:

- **API key** - a dedicated key issued to the agent, sent in a header (e.g. `x-api-key`). Simple; Sunbay controls issuance and rotation.
- **OAuth 2.0 client credentials** - the agent exchanges a client id/secret for a short-lived bearer token. More standard and rotatable.
- **Mutual TLS (mTLS)** - the agent is identified by a client certificate.

### 7.3 Tenant identification

Each call must let Sunbay determine **which end-client (tenant)** the data belongs to - via a `tenantCode` together with the credential, which Sunbay maps to a single tenant. Data is isolated per tenant; a credential can only write data for its own tenant.

> Because the agent runs on-premise with **dynamic/unknown outbound IPs**, IP allowlisting is generally impractical - authentication relies on the credential above.

### 7.4 Data protection

- Encrypted in transit (TLS) and at rest on the Sunbay side.
- Sensitive values (keys, tokens) are never logged.
- Credentials are exchanged securely and can be rotated.

---

## 8. Reliability

- **Idempotency.** `externalInvoiceId` is the key: re-sending the same invoice is a safe **update**, never a duplicate. The agent **must not** change an invoice's `externalInvoiceId` between sends/retries.
- **Retries.** On a transient/network/`5xx` failure the agent should retry with backoff; a malformed/`4xx` request should not be blindly retried. The precise retry contract is tied to the acknowledgment design (§4.4) and agreed at onboarding.
- **Ordering.** Corrective documents reference the original via `correctedExternalInvoiceId`. If a correction can be sent before its original, we agree how to handle it (hold / accept-then-link).

---

## 9. Scheduling & Volume

- **Interval.** The agent pushes on a configurable schedule per client (e.g. every 15 minutes, hourly, or daily), set during onboarding.
- **Volume.** With the per-invoice model, call volume scales with the number of invoices per cycle. Please share expected **daily and peak volumes** (e.g. month-end). For very high volumes, the bulk data-only path (§2.2) plus per-invoice PDFs for open invoices only is recommended.
- **Rate limits.** Sunbay may apply rate limiting; limits and any `Retry-After` behaviour are agreed so the agent's pace fits within them.

---

## 10. Open Questions for Emerge

> **Note:** these questions are for the implementation/rollout phase itself - not something to resolve now. They do not block reviewing or sharing this document; we work through them together when the integration is actually being built.

These depend on how the agent is implemented and on each ERP; we'll resolve them together during onboarding.

1. **Stable identifiers** - Does each ERP (Comarch, Enova, Symfonia) expose a stable, unique id per **invoice** and per **customer** that survives edits? What are they?
2. **Corrections (korekta)** - How does each ERP represent a corrective document, and are its amounts the **new corrected totals**, the **delta**, or "before/after"?
3. **Partial payments** - Can the agent provide `amountPaid` / `amountOutstanding`, or only a binary paid flag?
4. **Cancellations/deletions** - In incremental mode, how do we learn an invoice was cancelled/deleted in the ERP (it won't show up as a "change")?
5. **Acknowledgment** - Does Emerge want to **read back** per-invoice/per-batch results? If so, what response bodies should Sunbay return, how should errors/duplicates be signalled, and how does this tie into retries? (§4.4)
6. **PDFs** - Always available? Maximum size? Are there PDFs for corrective documents? One PDF per invoice always?
7. **Scope** - Sales/revenue invoices only (we do not chase purchase invoices). Should proformas be excluded? Which document types are "collectible"?
8. **Multi-company** - Can one installation hold several legal entities/NIPs? If so, how is the seller/tenant disambiguated?
9. **Currencies** - Are multi-currency invoices expected?
10. **Backfill** - How much history on the first load (all open, plus paid within N months)? Via bulk data-only or per-invoice?
11. **Schedule & time** - Push frequency per client; time-zone handling for dates.
12. **Formats** - Confirm UTF-8, dot decimals, ISO-8601, phone with country code can be guaranteed by the agent (§6).
13. **Authentication** - Preferred method (§7.2) and feasibility from the on-premise environment.
14. **Volume** - Expected daily and peak invoice counts per client.

---

*Prepared by Sunbay for the Emerge integration. Questions: integrations@sunbay.io*
