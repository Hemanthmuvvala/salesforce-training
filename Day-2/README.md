# Day 2: Apex Triggers & Governor Limits

> **Salesforce Platform Developer Bootcamp — Day 2**

This document covers the core concepts, design decisions, and learnings from Day 2, which focused on writing Apex Triggers, respecting Governor Limits, and building bulkified, production-ready code.

---

## Table of Contents

1. [Why Did You Choose a Trigger?](#1-why-did-you-choose-a-trigger)
2. [Why Before Insert?](#2-why-before-insert)
3. [How Did You Bulkify Your Code?](#3-how-did-you-bulkify-your-code)
4. [What Did You Learn Today?](#4-what-did-you-learn-today)

---

## 1. Why Did You Choose a Trigger?

A **Trigger** was chosen because the automation requirement was tightly coupled to data persistence events — specifically, the need to intercept and modify records **before** they are committed to the database, or to perform logic **after** a DML operation in response to record changes.

Declarative tools like **Flow** or **Process Builder** were evaluated first (following Salesforce's recommended "declarative-first" approach), but they were ruled out for the following reasons:

- **Complex conditional logic** that required procedural programming constructs (loops, maps, sets) not expressible in Flow.
- **Cross-object data manipulation** that needed precise control over field values across multiple related records in a single transaction.
- **Performance requirements** — Apex Triggers execute within a single transaction context, giving deterministic, low-latency execution compared to asynchronous Flow paths.
- **Governor Limit awareness** — writing Apex allowed explicit control over SOQL query placement, DML statement batching, and heap usage, which declarative tools abstract away, sometimes unpredictably.

Triggers provide the lowest-level hook into the Salesforce record lifecycle, making them the correct tool when business logic **must** be synchronous, transactional, and bulk-safe.

---

## 2. Why Before Insert?

The **`before insert`** trigger context was selected for the following architectural reasons:

| Factor | Explanation |
|--------|-------------|
| **No DML Required for Updates** | In a `before` context, modifications to `Trigger.new` records are applied directly to the in-memory sObject list. Salesforce writes these changes to the database automatically, eliminating the need for an additional `update` DML call. |
| **Governor Limit Efficiency** | Avoiding an extra `update` DML statement saves one of the 150 DML statements allowed per transaction, keeping the code well within limits. |
| **Data Integrity** | Validation and field defaulting logic runs before the record is saved, ensuring no invalid or incomplete record ever reaches the database. |
| **No Re-trigger Risk** | Since no explicit DML is issued on the same object inside the trigger, there is no risk of accidentally re-firing the trigger (recursive execution). |

A `before insert` trigger is the canonical Salesforce pattern for **field population, defaulting, and pre-save validation** — exactly the use case encountered on Day 2.

---

## 3. How Did You Bulkify Your Code?

Bulkification is a fundamental Salesforce best practice. Since a single save action can affect up to **200 records** at once (e.g., a data import), all trigger logic was written to handle collections rather than individual records.

### Key Bulkification Techniques Applied

**1. Iterate Over `Trigger.new` Instead of Assuming a Single Record**
```apex
for (Opportunity opp : Trigger.new) {
    // Process every record in the batch, not just one
}
```

**2. Collect IDs First, Then Query Outside the Loop**

Performing SOQL inside a `for` loop is the most common Governor Limit violation. All required data was collected into a `Set<Id>` first, then fetched in a single query.

```apex
Set<Id> accountIds = new Set<Id>();
for (Opportunity opp : Trigger.new) {
    accountIds.add(opp.AccountId);
}

Map<Id, Account> accountMap = new Map<Id, Account>(
    [SELECT Id, Industry, Rating FROM Account WHERE Id IN :accountIds]
);
```

**3. Use Maps for O(1) Lookups**

A `Map<Id, SObject>` was used to retrieve related records in constant time during the iteration loop, rather than issuing per-record queries.

```apex
for (Opportunity opp : Trigger.new) {
    Account relatedAccount = accountMap.get(opp.AccountId);
    if (relatedAccount != null) {
        // Apply field logic using the pre-fetched account
    }
}
```

**4. Batch DML Statements**

All records requiring updates were collected into a `List<SObject>` and committed with a single DML statement outside the loop.

```apex
List<Task> tasksToInsert = new List<Task>();
for (Opportunity opp : Trigger.new) {
    tasksToInsert.add(new Task(/* ... */));
}
if (!tasksToInsert.isEmpty()) {
    insert tasksToInsert;
}
```

### Governor Limits Respected

| Limit | Limit Value | How It Was Respected |
|-------|-------------|----------------------|
| SOQL Queries per Transaction | 100 | All queries moved outside loops |
| DML Statements per Transaction | 150 | Batched into single DML calls |
| DML Rows per Transaction | 10,000 | Collections processed in bulk |
| Heap Size | 6 MB (sync) | Maps and Lists used efficiently |

---

## 4. What Did You Learn Today?

Today was a pivotal shift from declarative configuration to **programmatic development**. The key takeaways were:

- **Triggers fire on collections, not single records.** This is the most important mental model shift. Every trigger must be written assuming 200 records could arrive simultaneously.
- **SOQL inside a loop is a critical anti-pattern.** It can cause `System.LimitException: Too many SOQL queries: 101` errors in production, often triggered by bulk data imports or API calls.
- **The `before` context is more efficient than `after` for field modifications.** Modifying `Trigger.new` directly avoids an extra round-trip DML statement.
- **Maps are your best friend in Apex.** `Map<Id, SObject>` enables efficient lookups after a single bulk query, replacing the need for nested loops or repeated queries.
- **Governor Limits are not obstacles — they are design constraints.** Writing code that respects these limits produces leaner, faster, and more scalable solutions.
- **Always check `Trigger.isInsert`, `Trigger.isUpdate`, etc.** Using context variables ensures the trigger logic executes only in the intended scenario, preventing unexpected side effects.

---

*Bootcamp Status: In Progress | Next: Day 3 — Validation Rules, Flows & Triggers*
