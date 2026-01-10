# BIR‑Compliant POS System – Development README (Philippines)

This README defines the **functional, technical, and compliance requirements** for developing a **Bureau of Internal Revenue (BIR)–accredited Point‑of‑Sale (POS) system** in the Philippines. It is intended for **software engineers, system architects, QA, and compliance teams**.

---

## 1. Purpose & Scope

This document ensures that the POS system:
- Can be **accredited by the BIR** and issued a **Permit to Use (PTU)**
- Generates all **mandatory BIR reports**
- Maintains a **complete audit trail**
- Supports **future Electronic Invoicing / Sales Reporting (EIS/ESRS)** integration

This applies to:
- Standalone POS systems
- POS connected to a server or CAS
- Cloud‑based POS with offline capability

---

## 2. Regulatory References (For Context)

The POS must comply with, but is not limited to:
- BIR Permit‑to‑Use (PTU) rules for POS / CRM
- Revenue Memorandum Orders (e.g., RMO 24‑2023)
- Electronic Invoicing & Sales Reporting regulations (RR 11‑2025)

> ⚠️ Regulations evolve. The system must be **configurable**, not hard‑coded.

---

## 3. Core POS Functional Requirements

### 3.1 Transaction Processing

The POS **MUST**:
- Generate **unique, sequential invoice/receipt numbers**
- Prevent deletion of finalized transactions
- Support sales, voids, returns, refunds, and discounts
- Timestamp all transactions (date + time)
- Identify cashier/user per transaction

---

### 3.2 Official Receipt / Invoice Output

Each receipt/invoice must contain:
- Registered business name
- Business address
- TIN
- VAT registration status
- PTU / Accreditation number
- Machine ID / Terminal ID
- Invoice / OR number
- Date and time
- Itemized sales
- VAT breakdown (VATable, VAT‑exempt, Zero‑rated)
- Discounts (SC, PWD, promos)
- Total amount due

Reprints must be clearly marked:
> **"REPRINT – ORIGINAL ISSUED"**

---

## 4. Mandatory BIR Reports (NON‑NEGOTIABLE)

### 4.1 X‑Reading Report (Interim)

**Purpose:** Shift or interim sales monitoring

**Requirements:**
- Accumulated totals
- Does NOT reset counters
- Can be generated multiple times per day

---

### 4.2 Z‑Reading Report (End‑of‑Day)

**Purpose:** Official daily closure

**Requirements:**
- Total gross sales
- VATable / VAT‑exempt / Zero‑rated sales
- VAT amount
- Discounts, voids, refunds
- Transaction count
- Beginning and ending invoice numbers
- Resets daily counters

⚠️ Once generated, Z‑Reading **CANNOT be edited or deleted**

---

### 4.3 Electronic Journal (E‑Journal)

**Purpose:** Full audit trail

**Must log:**
- All transactions
- Voids, refunds, returns
- Reprints
- User logins/logouts
- System actions affecting sales data

**Rules:**
- Append‑only
- Non‑volatile
- Tamper‑proof

---

### 4.4 BIR Sales Summary Report

**Contents:**
- Reporting period
- Gross sales
- Net sales
- VAT details
- Discounts & deductions
- Z‑counter reference

Used for:
- BIR audits
- VAT filings

---

### 4.5 Supporting Reports

The POS **MUST** also generate:
- Detailed Sales Transaction Report
- Refund / Return Report
- Discount Report (SC, PWD, Others)
- Cashier / User Activity Report

---

## 5. Data Integrity & Security Requirements

### 5.1 Audit & Tamper Protection

- No hard deletion of sales records
- All changes logged with:
  - Who
  - When
  - What changed
- Role‑based access control

---

### 5.2 Data Storage

- Persistent storage (local + backup)
- Data retention support (minimum 10 years)
- Offline mode with sync on reconnect

---

## 6. System Architecture Requirements

### 6.1 Identification

Each POS instance must have:
- Unique Terminal ID
- Machine Serial Number (logical or physical)
- Store / Branch ID

---

### 6.2 Configuration Controls

The system must allow:
- VAT rate configuration
- Discount rule configuration
- Receipt layout updates (without code changes)

---

## 7. Electronic Invoicing / Sales Reporting (Future‑Proofing)

The POS should be designed to:
- Export invoices in **structured format (JSON/XML)**
- Support API‑based submission to BIR EIS
- Maintain transmission status (Pending / Sent / Failed)
- Retry failed submissions

> Even if not required immediately, **architecture must support this**.

---

## 8. Accreditation & Deployment Considerations

Before production use:
- POS must be registered via **eAccReg**
- PTU must be issued per branch / terminal
- Sample reports must be reproducible

Any change to:
- POS software logic
- Report format
- Tax computation

➡️ **May require re‑notification or re‑accreditation**

---

## 9. Development Do‑Not‑Dos (Critical)

🚫 No editing of Z‑Read data
🚫 No resetting invoice numbers
🚫 No deleting finalized transactions
🚫 No silent data correction

Violations may cause:
- PTU revocation
- BIR penalties
- Criminal liability for users

---

## 10. Recommended Development Checklist

- [ ] Sequential invoice engine
- [ ] X‑Reading report
- [ ] Z‑Reading report
- [ ] E‑Journal
- [ ] Sales summary reports
- [ ] Audit logs
- [ ] Offline resilience
- [ ] Configurable tax rules
- [ ] Structured data export

---

## 11. Final Notes

This README is a **baseline**. Actual BIR accreditation depends on:
- Submitted documentation
- Sample outputs
- RDO interpretation

Design conservatively. **Compliance > Convenience.**

---

**End of Document**

