# Energy CRM Onboarding Automation (n8n, Gmail, Airtable)

Automates customer onboarding for an energy CRM: receives customer data via webhook, validates required fields, creates an Airtable customer record when valid, and sends an email notification when invalid.

---

## Workflow

<img width="1037" height="411" alt="image" src="https://github.com/user-attachments/assets/adcc997e-63f1-4145-8b39-2ff61acae790" />


## Features

- Webhook-based ingestion (JSON via HTTP POST)
- Data validation with clear output (`isValid`, `missingFields`, normalized `data`)
- Branching logic:
  - **TRUE** → Create customer in Airtable
  - **FALSE** → Send Gmail alert with missing fields + payload
- Clean mappings and repeatable test payloads

---

## Tech stack

- **n8n** (workflow automation) & Docker (Self-hosted n8n)
- **Airtable** (CRM database)
- **Postman** (API tools)
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
- Base: Energy CRM
- Table: Customers
- Create data from "Data folder"
<img width="700" height="500" alt="image" src="https://github.com/user-attachments/assets/65c818e5-c1b3-42a2-a2cd-f88c6f1f374e" />

### Setup
1. Create API Token at Airtable - https://airtable.com/create/tokens/new
2. Activate scopes:
   - data.records:read 
   - data.records:write 
   - schema.bases:read 
4. Access -> Add database "Energy CRM" & get the token (for n8n)
5. Create Airtable credential in n8n
   
   <img width="400" height="300" alt="image" src="https://github.com/user-attachments/assets/1f2eeddc-e078-4d86-8711-4bfffa49fa10" />
6. Add the token on API Key
   
   <img width="400" height="300" alt="image" src="https://github.com/user-attachments/assets/95e4a7f9-d948-4289-adc5-e5f28f61abec" />
   
7. Use / Create data for CRM list (from data)
   
   <img width="500" height="600" alt="image" src="https://github.com/user-attachments/assets/bd3352ba-b26a-4b1c-a9ab-67ef84c9f40a" />

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

## n8n setup
1) Webhook node
 - Method: POST
 - Content-Type expected: application/json

## Postman API - Testing 
- Go to https://web.postman.co/ -> Workspace and create new HTTP
-	Change GET to POST
-	Add the URL from n8n (http://localhost:5678/webhook-test/crm-customer-onboarding)
-	Add Information at Param “Content-Type” and Value: “application/json”

<img width="945" height="321" alt="image" src="https://github.com/user-attachments/assets/fe641194-9a3e-4869-a7c3-645568f15038" />

-	Go to Body and use JSON and copy the JSON test

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

-	Execute the workflow on N8N then click Send on Postman after you will see this 200 OK, means working and the results below

<img width="945" height="478" alt="image" src="https://github.com/user-attachments/assets/b7596072-d6a5-4364-8481-2f66174b559b" />


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
The result looks like this 

<img width="945" height="387" alt="image" src="https://github.com/user-attachments/assets/8bfd7072-54e0-4ff6-b3c0-38ff07d57631" />

3) IF node (routing)
Configure one condition only:
- Value 1 (Expression): {{ $json.isValid }}
- Operation: is equal to
- Value 2: true (Boolean)

Result:
- true → TRUE output
- false → FALSE output

4) Airtable node (TRUE branch)
Node: Airtable → Create Record → Create

Setup:
a.	Credential connect to Airtable account with connected API
b.	Resource -> Record
c.	Operation: Create
d.	Base -> From List -> Energy CRM
e.	Table -> From List -> Customers
f.	Mappig Column Mode -> Map Each Column Manually -> then appear “Values to Send” with those columns from Airtable
g.	IF data must appear and move the data as Expression into each column

Map fields from the normalized object:
- Company_Name → {{ $json.data.company_name }}
- Contact_Person → {{ $json.data.contact_person }}
- Email → {{ $json.data.email }}
- Phone → {{ $json.data.phone }}
- Address → {{ $json.data.address }}
- Customer_Type → {{ $json.data.customer_type }}
- Notes → Customer onboarded via webhook API
  
Optional fixed values (recommended):
- Onboarding_Status → New Lead
- Assigned_To → Sarah Schmidt

<img width="945" height="382" alt="image" src="https://github.com/user-attachments/assets/5cdf4eb7-55b3-43fa-bc1f-d8af408553b2" />

5) Gmail node (FALSE branch)
Node: Gmail → Send a Message → Send

Setup
a. Conditions use Expression and add {{ $json.isValid }} -> is equal to (Boolean) -> true
b. Google Cloud -> Enable Gmail API
  i.	Create OAuth Client ID
  ii.	Application Type: Web application
  iii.	Name: From the project (free to choose)
  iv.	Authorized redirect URIs (from n8n)

  <img width="400" height="426" alt="image" src="https://github.com/user-attachments/assets/b512fde6-065e-4ca1-a97d-8d23219a5701" />

  <img width="801" height="244" alt="image" src="https://github.com/user-attachments/assets/9e84840d-135e-4650-9272-4f89434e7b91" />

c. Add the "Client ID" and "Client secret" information into n8n

<img width="945" height="444" alt="image" src="https://github.com/user-attachments/assets/e9e2f891-b559-4d17-a965-26682d061ba3" />



## How to test
1. Valid payload (should go TRUE → Airtable)
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

2. Invalid payload (should go FALSE → Gmail)
- Go to POSTMAN API and delete JSON info of "customer type": "Solar"
  
```json
{
  "company_name": "Test Solar GmbH",
  "email": "test@testsolar.de",
  "contact_person": "Max Mustermann",
  "phone": "+49 30 12345678",
  "address": "Teststraße 1, 10115 Berlin"
}
```

Result:
- TRUE test: record appears in Airtable Customers


- FALSE test: alert email arrives (also check Spam/Promotions and the sender’s “Sent” folder)
- 
<img width="933" height="641" alt="image" src="https://github.com/user-attachments/assets/c8ef6438-e293-4691-a826-c9264fe8a60f" />



