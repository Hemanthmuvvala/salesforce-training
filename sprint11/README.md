# Sprint 11 — Crossing the Salesforce Boundary
### REST API Integration | Hospital Management System × Pharmacy Network

---

## 🏥 Business Problem

The hospital's **Doctor → Appointment → Prescription** workflow is fully managed inside Salesforce. However, when a doctor creates a prescription, the external **Pharmacy Network** has no visibility — medications aren't prepared, and patients face delays at pickup.

**Solution:** Automatically sync every new prescription to the external Pharmacy Network REST API the moment it is created in Salesforce, without blocking the core save transaction.

---

## 🏗️ Integration Architecture

The integration follows a **trigger-driven, asynchronous** pattern using Queueable Apex. Salesforce commits its own data first; the external callout happens independently in a separate transaction.

```
Prescription__c (After Insert Trigger)
        │
        ▼
PrescriptionService.cls  ──► enqueues job
        │
        ▼
PrescriptionSyncQueueable.cls
        │
        ▼
HTTP POST  ──►  callout:Pharmacy_Network_API/prescriptions
        │
        ▼
Update Prescription__c  (Integration_Status__c, External_Prescription_Id__c)
```

> See `architecture/integration-flow.png` for the full flow diagram,  
> `architecture/sequence-diagram.png` for the sequence diagram, and  
> `architecture/integration-pattern.png` for the integration pattern overview.

---

## 🔐 Security — Zero Hard-Coding Rule

| What | How |
|---|---|
| Base URL & endpoint | **Named Credential** — `callout:Pharmacy_Network_API` |
| Bearer Token / OAuth | **Auth Provider** attached to the Named Credential |
| No secrets in Apex | ✅ Enforced — no hardcoded URLs or tokens anywhere |

---

## 📊 State Tracking Fields on `Prescription__c`

| Field API Name | Type | Values / Purpose |
|---|---|---|
| `Integration_Status__c` | Picklist | `Pending` · `Sent` · `Failed` · `Retry Required` |
| `External_Prescription_Id__c` | Text | ID returned by the Pharmacy API on success |
| `Last_Integration_Attempt__c` | DateTime | Timestamp of the most recent callout attempt |
| `Integration_Error__c` | Long Text | Raw error message / HTTP response body on failure |

---

## ⚙️ Error Handling Summary

| HTTP Status | Meaning | Action Taken |
|---|---|---|
| `200 / 201` | Success | Mark `Sent`, store external ID |
| `400` | Bad Request | Mark `Failed`, log error body |
| `401 / 403` | Auth Issue | Mark `Failed`, alert admin |
| `500` | Server Error | Mark `Retry Required`, re-enqueue |

Idempotency is guaranteed by sending the Salesforce **Prescription Record ID** as `prescriptionRef` — the Pharmacy API rejects duplicate submissions.

---

## 🚀 Setup Instructions

### 1. Deploy Metadata
```bash
sf project deploy start --source-dir force-app
```

### 2. Create Named Credential
- **Setup → Named Credentials → New**
- Label: `Pharmacy Network API`
- Name: `Pharmacy_Network_API`
- URL: *(provided by Pharmacy partner)*
- Identity Type: `Named Principal`
- Auth Protocol: Attach your OAuth **Auth Provider**

### 3. Add Remote Site Setting *(if required)*
- **Setup → Remote Site Settings → New**
- Add the Pharmacy API base URL

### 4. Configure Custom Fields
Ensure all four tracking fields exist on `Prescription__c` (see table above). Deploy via included metadata in `force-app/`.

### 5. Test the Integration
1. Create a **Doctor**, **Patient**, and **Appointment** record.
2. Create a **Prescription** linked to the Appointment.
3. After save, check `Integration_Status__c` — it should move from `Pending` → `Sent`.
4. Verify `External_Prescription_Id__c` is populated.

---

## 📁 Repository Structure

```
sprint11/
├── README.md                     ← You are here
├── architecture/
│   ├── integration-flow.png      ← End-to-end flow diagram
│   ├── sequence-diagram.png      ← UML sequence diagram
│   └── integration-pattern.png   ← P2P integration pattern
├── force-app/                    ← Salesforce source metadata
├── api-contract/
│   └── pharmacy-api.md           ← Formal API contract
├── screenshots/                  ← Demo & verification screenshots
└── learning-notes/
    └── sprint-11.md              ← Key engineering concepts
```

---

## 📚 Further Reading
- [API Contract](api-contract/pharmacy-api.md)
- [Learning Notes — Sprint 11](learning-notes/sprint-11.md)
- [Salesforce Queueable Apex Docs](https://developer.salesforce.com/docs/atlas.en-us.apexcode.meta/apexcode/apex_queueing_jobs.htm)
- [Named Credentials Docs](https://help.salesforce.com/s/articleView?id=sf.named_credentials_about.htm)
