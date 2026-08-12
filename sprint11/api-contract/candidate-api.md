# API Contract — Pharmacy Network Integration

**Specification Version:** 1.0  
**Sprint:** 11 — Crossing the Salesforce Boundary  
**Last Updated:** 2026-08-12  
**Maintained By:** Salesforce Integration Team

---

## Endpoint

| Property | Value |
|---|---|
| **Method** | `POST` |
| **Path** | `/prescriptions` |
| **Full Callout URL** | `callout:Pharmacy_Network_API/prescriptions` |
| **Content-Type** | `application/json` |
| **Authentication** | OAuth 2.0 Bearer Token via Named Credential Auth Provider |

---

## Authentication

All requests are authenticated via **OAuth 2.0 Client Credentials** flow, managed entirely by the Salesforce **Named Credential** (`Pharmacy_Network_API`) and its attached **Auth Provider**.

- ✅ No tokens or secrets are written in Apex code.
- The platform automatically injects the `Authorization: Bearer <token>` header.
- Token refresh is handled automatically by the Auth Provider.

---

## Request Body

### Schema

| Field | Type | Required | Description |
|---|---|---|---|
| `prescriptionRef` | `string` | ✅ | Salesforce Prescription Record ID — used for idempotency |
| `patientId` | `string` | ✅ | Hospital Patient identifier |
| `patientName` | `string` | ✅ | Full name of the patient |
| `doctorId` | `string` | ✅ | Hospital Doctor identifier |
| `doctorName` | `string` | ✅ | Full name of the prescribing doctor |
| `medication` | `string` | ✅ | Medication name and strength (e.g., `Amoxicillin 500mg`) |
| `dosage` | `string` | ✅ | Full dosage instruction |
| `issueDate` | `string` | ✅ | Date of prescription in `YYYY-MM-DD` format |

### Example Payload

```json
{
  "prescriptionRef": "PRX-55092",
  "patientId": "PAT-10045",
  "patientName": "John Doe",
  "doctorId": "DOC-702",
  "doctorName": "Dr. Sarah Smith",
  "medication": "Amoxicillin 500mg",
  "dosage": "1 tablet twice a day for 7 days",
  "issueDate": "2026-08-12"
}
```

---

## Responses

### ✅ Success — `201 Created`

```json
{
  "status": "success",
  "externalPrescriptionId": "PHM-EXT-98234",
  "message": "Prescription received and queued for dispensing."
}
```

> On success: store `externalPrescriptionId` in `External_Prescription_Id__c` and set `Integration_Status__c = Sent`.

### ✅ Idempotent Hit — `200 OK`

```json
{
  "status": "already_exists",
  "externalPrescriptionId": "PHM-EXT-98234",
  "message": "Prescription with this reference already exists."
}
```

---

## Error Responses

| HTTP Code | Scenario | Salesforce Action |
|---|---|---|
| `400 Bad Request` | Missing or invalid fields | Set `Failed`, log response body in `Integration_Error__c` |
| `401 Unauthorized` | Token missing or expired | Set `Failed`, alert admin — re-check Auth Provider |
| `403 Forbidden` | Insufficient permissions | Set `Failed`, escalate to Pharmacy partner |
| `409 Conflict` | Duplicate detected (fallback) | Set `Sent` — idempotent, already processed |
| `500 Server Error` | Transient pharmacy system error | Set `Retry Required`, re-enqueue Queueable job |

### Error Response Body (Example)

```json
{
  "status": "error",
  "code": "INVALID_PAYLOAD",
  "message": "Field 'medication' is required and cannot be blank."
}
```

---

## Idempotency Strategy

| Layer | Mechanism |
|---|---|
| **Request level** | `prescriptionRef` (Salesforce Record ID) is sent on every request. The Pharmacy API uses this as a unique key and returns `200` instead of creating a duplicate. |
| **Salesforce level** | Before enqueuing, the service checks `Integration_Status__c != 'Sent'` to avoid redundant callouts. |
| **Result** | Even if the Queueable job runs twice (e.g., platform retry), no duplicate medication order is created in the Pharmacy system. |

---

## Apex Callout Reference

```apex
HttpRequest req = new HttpRequest();
req.setEndpoint('callout:Pharmacy_Network_API/prescriptions');
req.setMethod('POST');
req.setHeader('Content-Type', 'application/json');
req.setBody(JSON.serialize(payload));

Http http = new Http();
HttpResponse res = http.send(req);
```

> **No hardcoded URLs or tokens.** The Named Credential resolves the base URL and injects the Bearer token automatically.
