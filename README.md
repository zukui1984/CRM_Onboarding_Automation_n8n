# Energy CRM Onboarding Automation (n8n + Airtable)

Automates customer onboarding for an energy CRM: receives customer data via webhook, validates required fields, creates an Airtable customer record when valid, and sends an email notification when invalid.

---

## Features

- Webhook-based ingestion (JSON via HTTP POST)
- Data validation with clear output (`isValid`, `missingFields`, normalized `data`)
- Branching logic:
  - **TRUE** → Create customer in Airtable
  - **FALSE** → Send Gmail alert with missing fields + payload
- Clean mappings and repeatable test payloads

---

## Tech stack

- **n8n** (workflow automation)
- **Airtable** (CRM database)
- **Gmail** (notifications via Gmail node)

---

## Workflow overview

```text
Webhook (POST)
  → Data Validation (Code)
  → IF (isValid == true)
       ├─ TRUE  → Airtable: Create Customer record
       └─ FALSE → Gmail: Send alert email (missing fields)
```

## Airtable data model
Base: Energy CRM
Table: Customers

Recommended fields (types in brackets):
- Company_Name (Single line text)
- Contact_Person (Single line text)
- Email (Email)
- Phone (Single line text)
- Address (Single line text or Long text)
- Customer_Type (Single select; e.g., Solar / Wind / Battery Storage / Hybrid)
- Onboarding_Status (Single select; e.g., New Lead / Documents Pending / Active)
- Assigned_To (Single select or collaborator)
- Notes (Long text)

Important: Make sure Email is an Email or text field (not Single select), otherwise Airtable may reject new email values.

## n8n setup
1) Webhook node
 - Method: POST
 - Content-Type expected: application/json

2) Data Validation (Code node)
Paste this code into the Code node:

```js
const incoming = $input.first().json;

// Supports webhook format { body: {...}, ... } or direct JSON
const payload = incoming.body ?? incoming;

// If nested again, unwrap one more time
const data = payload.body ?? payload;

const required = ['company_name', 'email', 'contact_person', 'customer_type'];
const missing = required.filter(
  (k) => !data?.[k] || String(data[k]).trim() === ''
);

return [
  {
    json: {
      isValid: missing.length === 0,
      missingFields: missing,
      data,
    },
  },
];
```

3) IF node (routing)
Configure one condition only:
- Value 1 (Expression): {{ $json.isValid }}
- Operation: is equal to
- Value 2: true (Boolean)

Result:
- true → TRUE output
- false → FALSE output

4) Airtable node (TRUE branch)
Node: Airtable → Record → Create

Map fields from the normalized object:
- Company_Name → {{ $json.data.company_name }}
- Contact_Person → {{ $json.data.contact_person }}
- Email → {{ $json.data.email }}
- Phone → {{ $json.data.phone }}
- Address → {{ $json.data.address }}
- Customer_Type → {{ $json.data.customer_type }}
  
Optional fixed values (recommended):
- Onboarding_Status → New Lead

Assigned_To → Sarah Schmidt

Notes → Customer onboarded via webhook API

5) Gmail node (FALSE branch)
Node: Gmail → Message → Send

Suggested config:
- To: internal ops/support email (e.g., ops@yourcompany.com)
- Subject (Expression):

```text
❌ Onboarding failed — missing: {{ $json.missingFields.join(', ') }}
```

- Email Type: Text (simpler for debugging) or HTML
- Message (Expression):

```text
Onboarding FAILED.

Missing fields:
{{ $json.missingFields.join(', ') }}

Received payload:
{{ JSON.stringify($json.data, null, 2) }}
```

## How to test
Valid payload (should go TRUE → Airtable)
```json
{
  "company_name": "Test Solar GmbH",
  "email": "test@testsolar.de",
  "contact_person": "Max Mustermann",
  "phone": "+49 30 12345678",
  "address": "Teststraße 1, 10115 Berlin",
  "customer_type": "Solar"
}
```

Invalid payload (should go FALSE → Gmail)
```json
{
  "company_name": "Test Solar GmbH",
  "email": "test@testsolar.de",
  "contact_person": "Max Mustermann",
  "phone": "+49 30 12345678",
  "address": "Teststraße 1, 10115 Berlin"
}
```

What to verify:
- TRUE test: record appears in Airtable Customers
- FALSE test: alert email arrives (also check Spam/Promotions and the sender’s “Sent” folder)

Production usage (automation)
1. Activate the workflow in n8n.
2. Use the production webhook URL in the real system that submits onboarding data (website form, internal tool, etc.).
3. n8n will route requests automatically:
 - valid → Airtable
 - invalid → Gmail
