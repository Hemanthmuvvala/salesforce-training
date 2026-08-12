# Day 6: Advanced Apex Triggers & Bulkification

> **Salesforce Platform Developer Bootcamp — Day 6**

This document details the professional trigger design patterns applied on Day 6 — specifically the **Trigger Handler pattern**, recursive execution prevention, and leveraging context variables for safe bulk operations.

---

## Table of Contents

1. [Why the Trigger Handler Pattern?](#1-why-the-trigger-handler-pattern)
2. [How Did You Prevent Recursive Trigger Execution?](#2-how-did-you-prevent-recursive-trigger-execution)
3. [How Did You Leverage Context Variables for Bulk Operations?](#3-how-did-you-leverage-context-variables-for-bulk-operations)
4. [What Did You Learn Today?](#4-what-did-you-learn-today)

---

## 1. Why the Trigger Handler Pattern?

The **Trigger Handler pattern** separates the thin trigger file (which only invokes a handler) from the actual business logic (which lives in a dedicated Apex class). This is the industry-standard approach used by professional Salesforce development teams.

### The Problem with Logic-in-Trigger

Writing business logic directly in a trigger file creates several long-term problems:

| Problem | Impact |
|---------|--------|
| **Untestable in isolation** | Apex test classes cannot instantiate a trigger — logic embedded in the trigger cannot be unit tested cleanly |
| **Unmaintainable** | Triggers grow into thousands of lines as requirements accumulate, making them hard to read and debug |
| **No separation of concerns** | Business logic, DML operations, and governor limit management are all mixed together |
| **No reusability** | Logic in a trigger file cannot be called from other classes, batch jobs, or schedulable classes |

### The Trigger Handler Solution

```
PlacementTrigger.trigger  (thin — 5-10 lines, only delegates)
         |
         v
PlacementTriggerHandler.cls  (all business logic lives here)
```

**`PlacementTrigger.trigger`** — The trigger file is kept deliberately minimal:

```apex
trigger PlacementTrigger on Placement__c (
    before insert, before update,
    after insert, after update, after delete
) {
    PlacementTriggerHandler handler = new PlacementTriggerHandler();

    if (Trigger.isBefore) {
        if (Trigger.isInsert) handler.onBeforeInsert(Trigger.new);
        if (Trigger.isUpdate) handler.onBeforeUpdate(Trigger.new, Trigger.oldMap);
    }
    if (Trigger.isAfter) {
        if (Trigger.isInsert) handler.onAfterInsert(Trigger.new);
        if (Trigger.isUpdate) handler.onAfterUpdate(Trigger.new, Trigger.oldMap);
        if (Trigger.isDelete) handler.onAfterDelete(Trigger.old);
    }
}
```

**`PlacementTriggerHandler.cls`** — All logic lives here, fully testable and organized by context:

```apex
public class PlacementTriggerHandler {

    public void onBeforeInsert(List<Placement__c> newList) {
        PlacementService.setDefaultStatus(newList);
        PlacementService.populateDaysToPlace(newList);
    }

    public void onAfterInsert(List<Placement__c> newList) {
        PlacementService.updateJobOpeningCounts(newList, null);
    }

    public void onAfterUpdate(List<Placement__c> newList, Map<Id, Placement__c> oldMap) {
        PlacementService.handleStatusChange(newList, oldMap);
        PlacementService.updateJobOpeningCounts(newList, oldMap);
    }
    // ...
}
```

**Benefits Achieved:**
- ✅ Business logic in `PlacementTriggerHandler` can be instantiated and tested directly.
- ✅ Each method has a single, clear responsibility.
- ✅ New requirements are added as new methods, not by bloating the trigger file.
- ✅ Handler methods can be reused by batch jobs and schedulable classes.

---

## 2. How Did You Prevent Recursive Trigger Execution?

**Recursive trigger execution** occurs when a trigger performs a DML operation on the same object it fires on, causing the trigger to fire again — potentially entering an infinite loop and hitting governor limits.

### The Root Cause

```
After Insert on Placement__c fires
  → Handler updates Placement__c records (DML)
    → After Update on Placement__c fires again
      → Handler updates again → infinite recursion
```

### Solution: Static Boolean Guard

A `static Boolean` variable on a handler or utility class is set to `true` when the trigger first executes. Subsequent recursive calls check this flag and exit immediately.

**`TriggerExecutionContext.cls`** — A dedicated recursion-prevention class:

```apex
public class TriggerExecutionContext {
    // Static variables are initialized once per transaction, not per trigger invocation
    public static Boolean isPlacementTriggerRunning = false;
}
```

**In the trigger handler:**

```apex
public void onAfterInsert(List<Placement__c> newList) {
    // Guard: if we're already inside this trigger, do not re-execute
    if (TriggerExecutionContext.isPlacementTriggerRunning) {
        return;
    }

    TriggerExecutionContext.isPlacementTriggerRunning = true;

    try {
        PlacementService.updateJobOpeningCounts(newList, null);
        // Any DML here will not re-trigger this block
    } finally {
        // Always reset so the flag doesn't leak into future operations in the same transaction
        TriggerExecutionContext.isPlacementTriggerRunning = false;
    }
}
```

### Why `static`?

Static variables in Apex are **transaction-scoped**, not instance-scoped. They persist their value across all invocations within a single transaction. When the trigger fires a second time (recursively), the same static variable is checked — it's already `true`, so the handler exits without executing, breaking the recursive loop.

### When the Flag Is Reset

The flag is reset in a `finally` block to ensure it reverts to `false` after the protected block completes, allowing the next legitimate trigger invocation (e.g., from a different user action later) to execute normally.

---

## 3. How Did You Leverage Context Variables for Bulk Operations?

Apex trigger context variables provide rich metadata about the current trigger invocation. Mastering `Trigger.new`, `Trigger.newMap`, `Trigger.old`, and `Trigger.oldMap` is essential for writing correct, efficient bulk trigger logic.

### Context Variable Reference

| Variable | Available In | Type | Description |
|----------|-------------|------|-------------|
| `Trigger.new` | Before/After Insert, Before/After Update | `List<SObject>` | New versions of all records being saved |
| `Trigger.newMap` | After Insert, Before/After Update | `Map<Id, SObject>` | Map of new records keyed by record ID |
| `Trigger.old` | Before/After Update, Before/After Delete | `List<SObject>` | Old versions of all records (before save) |
| `Trigger.oldMap` | Before/After Update, Before/After Delete | `Map<Id, SObject>` | Map of old records keyed by record ID |

### Pattern 1: Detecting Changed Records Using `Trigger.oldMap`

Only process records where a specific field has actually changed — avoids running expensive logic on every update.

```apex
public void onAfterUpdate(List<Placement__c> newList, Map<Id, Placement__c> oldMap) {
    List<Placement__c> statusChangedPlacements = new List<Placement__c>();

    for (Placement__c newRecord : newList) {
        Placement__c oldRecord = oldMap.get(newRecord.Id);

        // Only process records where Status actually changed
        if (newRecord.Status__c != oldRecord.Status__c) {
            statusChangedPlacements.add(newRecord);
        }
    }

    if (!statusChangedPlacements.isEmpty()) {
        PlacementService.createStatusAuditLogs(statusChangedPlacements, oldMap);
    }
}
```

### Pattern 2: Aggregating Parent Record Updates with `Trigger.new`

Collect all parent IDs from the batch, then update parent records in a single DML call.

```apex
public void onAfterInsert(List<Placement__c> newList) {
    // Collect all unique Job Opening IDs from the batch
    Set<Id> jobOpeningIds = new Set<Id>();
    for (Placement__c p : newList) {
        if (p.Job_Opening__c != null) {
            jobOpeningIds.add(p.Job_Opening__c);
        }
    }

    // Count active placements per Job Opening in one aggregate query
    Map<Id, Integer> countMap = new Map<Id, Integer>();
    for (AggregateResult ar : [
        SELECT Job_Opening__c, COUNT(Id) cnt
        FROM Placement__c
        WHERE Job_Opening__c IN :jobOpeningIds
        AND Status__c != 'Withdrawn'
        GROUP BY Job_Opening__c
    ]) {
        countMap.put((Id)ar.get('Job_Opening__c'), (Integer)ar.get('cnt'));
    }

    // Build the update list and perform a single DML
    List<Job_Opening__c> jobsToUpdate = new List<Job_Opening__c>();
    for (Id jobId : countMap.keySet()) {
        jobsToUpdate.add(new Job_Opening__c(
            Id = jobId,
            Active_Placements__c = countMap.get(jobId)
        ));
    }
    if (!jobsToUpdate.isEmpty()) {
        update jobsToUpdate;
    }
}
```

### Pattern 3: Using `Trigger.newMap` for Quick Lookups Within the Same Batch

When one record in the batch needs to reference another record in the same batch (same transaction), `Trigger.newMap` enables O(1) lookups without a SOQL query.

```apex
// Find all records whose Parent_Placement__c points to another record in the same batch
for (Placement__c p : Trigger.new) {
    if (p.Parent_Placement__c != null && Trigger.newMap.containsKey(p.Parent_Placement__c)) {
        Placement__c parentInBatch = Trigger.newMap.get(p.Parent_Placement__c);
        // Cross-reference two records in the same transaction without a query
    }
}
```

---

## 4. What Did You Learn Today?

- **The Trigger Handler pattern is non-negotiable in professional Salesforce development.** No production org should have business logic written directly inside a trigger file. The maintainability and testability gains are too significant to ignore.
- **Static Booleans are the standard recursion guard.** They are transaction-scoped, cheap, and reliable. Always reset them in a `finally` block to avoid leaking state.
- **`Trigger.oldMap` is what makes update triggers intelligent.** Without it, you would have to process every record on every update. With it, you can filter to only the records where relevant data actually changed — a major performance win.
- **`Trigger.newMap` removes the need for lookups within the same batch.** When records in the same DML batch reference each other, `Trigger.newMap` provides instant access without any additional queries.
- **Aggregate SOQL queries (`GROUP BY`, `COUNT()`) replace loops that count records.** Instead of loading all child records and counting in Apex, a single aggregate query does the same work in the database, far more efficiently.
- **One trigger per object is a firm rule.** Multiple triggers on the same object fire in an unpredictable order. Consolidate all logic into one trigger file with a single handler class.

---

*Bootcamp Status: In Progress | Next: Day 7*
