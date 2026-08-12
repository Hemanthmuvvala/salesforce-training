# Sprint 11 — Learning Notes
### Crossing the Salesforce Boundary: REST API Integrations

**Date:** 2026-08-12  
**Domain:** Hospital Management System — Prescription Sync to Pharmacy Network

---

## 1. Authentication vs. Authorisation

These two terms are often confused but serve entirely different purposes.

| Concept | Question It Answers | Example in This Sprint |
|---|---|---|
| **Authentication** | *"Who are you?"* — Verify identity | Salesforce proves its identity to the Pharmacy API using an OAuth 2.0 Client Credentials token |
| **Authorisation** | *"What are you allowed to do?"* — Enforce permissions | Once authenticated, the Pharmacy API checks if Salesforce has permission to `POST` to `/prescriptions` |

> 💡 **Key takeaway:** You can be authenticated but still not authorised. A valid token with wrong scopes gives you a `403 Forbidden`, not a `401 Unauthorized`.

In Salesforce, the **Named Credential + Auth Provider** combo handles Authentication automatically. Authorisation is configured on the Pharmacy partner's side.

---

## 2. Point-to-Point vs. Middleware Integration

### Point-to-Point (This Sprint)
Salesforce connects **directly** to one external system — the Pharmacy Network API.

```
Salesforce  ──── HTTP POST ────►  Pharmacy Network API
```

**Pros:** Simple, fast to build, low overhead for a single partner.  
**Cons:** Does not scale. Every new partner = new Apex class, new Named Credential, new error handling logic.

### The Middleware / Hub-and-Spoke Problem
Imagine the hospital now needs to integrate with **50 systems**: labs, insurance providers, radiology, 30 different pharmacies.

```
                     ┌─────────────────┐
                     │   Middleware    │
                     │  (MuleSoft /    │
                     │   Boomi / etc.) │
                     └────────┬────────┘
          ┌──────────┬────────┴────────┬──────────┐
          ▼          ▼                 ▼          ▼
       Lab API   Insurance        Pharmacy 1   Pharmacy 2 ...
```

Salesforce sends **one message** to the middleware hub; the hub routes, transforms, and delivers to all relevant systems. Adding a new partner = configure in the hub only — **no Salesforce code changes**.

> 💡 **Key takeaway:** Point-to-Point is fine for 1–2 integrations. Beyond that, a middleware platform (MuleSoft, Azure Service Bus, Boomi) is essential to avoid a tangled "spaghetti integration" mess.

---

## 3. Copy Data vs. Access Data (External Objects)

### Copy Data (Standard Integration — This Sprint)
Prescription data is **copied** into the Pharmacy system. Salesforce stores its own copy; the Pharmacy stores its own copy.

- ✅ Fast reads (local data)
- ✅ Works offline
- ❌ Data duplication
- ❌ Risk of data drift (copies get out of sync)

### Access Data (Salesforce Connect / External Objects)
Instead of copying, Salesforce can **virtually display** data that lives in an external system using **External Objects** (via OData or custom adapters).

**Hospital Use Case:** The hospital wants to view a patient's **medical history from a national health registry** without importing thousands of records into Salesforce storage.

```
Salesforce UI  ──── OData Query ────►  National Registry API
                ◄─── Live Data ──────
```

- ✅ No data duplication, no storage cost
- ✅ Always shows live, real-time data
- ❌ Requires a live API connection (slower than local)
- ❌ Cannot run SOQL joins with standard objects easily

> 💡 **Key takeaway:** Use **Copy Data** when you need fast queries, offline access, or to run business logic on the data. Use **Access Data / External Objects** when the data belongs to an external system, is large, and you only need to *view* it occasionally.

---

## 4. Why Callouts and DML Must Be Separated

This is one of the most critical Salesforce governor limit rules for integration.

### The Rule
> ❌ You **cannot make a callout** after a **DML operation** in the **same transaction**.

### Why?
Salesforce holds a **database lock** after a DML statement. If you then make an HTTP callout (which can take seconds), the lock is held the entire time — blocking other transactions and risking deadlocks across the platform.

### How We Solved It in This Sprint

```
Transaction 1 (Trigger):
  ├── DML: Insert Prescription__c     ← Salesforce data saved ✅
  └── System.enqueueJob(...)          ← Queueable job registered, NO callout yet

Transaction 2 (Queueable — separate transaction):
  ├── HTTP Callout to Pharmacy API    ← No DML has happened yet ✅
  └── DML: Update Prescription__c     ← Update status AFTER callout ✅
```

By using **Queueable Apex**, the callout runs in its own transaction — completely separate from the trigger's DML. This satisfies the governor limit and also makes the integration **asynchronous**, so the user's save action is never blocked waiting for an external HTTP response.

> 💡 **Key takeaway:** Queueable Apex is the standard Salesforce pattern for "do DML now, make the callout later." Never mix callouts and DML in the same synchronous transaction.

---

## Summary

| Concept | Sprint 11 Application |
|---|---|
| Authentication vs Authorisation | Named Credential = Auth; Pharmacy API scope = Authz |
| Point-to-Point vs Middleware | Direct HTTP callout (P2P); middleware needed at scale |
| Copy vs Access Data | Prescription is copied; medical registries can use External Objects |
| Callout + DML separation | Trigger does DML → Queueable does callout + status update |
