# Day 8: Asynchronous Apex

> **Salesforce Platform Developer Bootcamp — Day 8**

This document covers background processing in Salesforce using **Asynchronous Apex** — the mechanisms for executing long-running operations outside the synchronous transaction, enabling callouts, large data processing, and scheduled jobs.

---

## Table of Contents

1. [Which Asynchronous Mechanism Was Selected and Why?](#1-which-asynchronous-mechanism-was-selected-and-why)
2. [How Do Governor Limits Compare: Async vs. Sync?](#2-how-do-governor-limits-compare-async-vs-sync)
3. [How Did You Handle State, Chaining & Callouts?](#3-how-did-you-handle-state-chaining--callouts)
4. [What Did You Learn Today?](#4-what-did-you-learn-today)

---

## 1. Which Asynchronous Mechanism Was Selected and Why?

Day 8 implemented two distinct asynchronous solutions, each chosen based on the characteristics of the use case.

### Use Case 1: External Job Board Callout — `@future`

**Mechanism Selected: `@future(callout=true)`**

When a Placement status changes to "Offer Extended", the system needs to notify an external job board API. This is a simple, fire-and-forget callout that does not need to chain to another operation or process thousands of records.

```apex
public class JobBoardIntegrationService {

    @future(callout=true)
    public static void notifyJobBoard(String placementId, String status) {
        HttpRequest req = new HttpRequest();
        req.setEndpoint('callout:Job_Board_API/placements/' + placementId);
        req.setMethod('PATCH');
        req.setHeader('Content-Type', 'application/json');
        req.setBody('{"status": "' + status + '"}');

        Http http = new Http();
        HttpResponse res = http.send(req);

        if (res.getStatusCode() != 200) {
            System.debug(LoggingLevel.ERROR, 'Job Board API call failed: ' + res.getBody());
        }
    }
}
```

**Why `@future`?**
- The callout is simple, one-directional, and doesn't need to return results or chain to another job.
- `@future` is the lightest-weight async option — ideal for offloading a single HTTP callout from a synchronous trigger.
- The `callout=true` annotation is required to allow HTTP calls from async context.

**Limitation acknowledged:** `@future` methods cannot be monitored in the Apex Jobs queue, cannot be chained, and cannot access `Trigger.new` state after the transaction — hence the explicit `String placementId` parameter.

---

### Use Case 2: Monthly Placement Report Generation — `Batch Apex` + `Schedulable`

**Mechanism Selected: `Database.Batchable` + `Schedulable`**

On the first day of every month, the system generates a summary report across all Placement records from the previous month — potentially tens of thousands of records. This exceeds synchronous limits and requires the job to run on a schedule.

```apex
// Batch class: processes large datasets in chunks of 200
public class MonthlyPlacementReportBatch implements Database.Batchable<sObject>, Database.Stateful {

    private Integer totalPlaced = 0;
    private Integer totalWithdrawn = 0;

    public Database.QueryLocator start(Database.BatchableContext bc) {
        Date startOfLastMonth = Date.today().toStartOfMonth().addMonths(-1);
        Date endOfLastMonth   = Date.today().toStartOfMonth().addDays(-1);

        return Database.getQueryLocator([
            SELECT Id, Status__c, Job_Opening__r.Department__c
            FROM Placement__c
            WHERE CreatedDate >= :startOfLastMonth
            AND   CreatedDate <= :endOfLastMonth
        ]);
    }

    public void execute(Database.BatchableContext bc, List<Placement__c> scope) {
        for (Placement__c p : scope) {
            if (p.Status__c == 'Placed')     totalPlaced++;
            if (p.Status__c == 'Withdrawn')  totalWithdrawn++;
        }
    }

    public void finish(Database.BatchableContext bc) {
        // Insert a summary report record after all chunks are processed
        Placement_Report__c report = new Placement_Report__c(
            Month__c         = Date.today().addMonths(-1).month(),
            Total_Placed__c  = totalPlaced,
            Total_Withdrawn__c = totalWithdrawn
        );
        insert report;
    }
}

// Schedulable class: fires the batch on a cron schedule
public class MonthlyPlacementReportScheduler implements Schedulable {
    public void execute(SchedulableContext sc) {
        MonthlyPlacementReportBatch batch = new MonthlyPlacementReportBatch();
        Database.executeBatch(batch, 200); // Process 200 records per chunk
    }
}
```

**Scheduling the job:**
```apex
// Run at midnight on the 1st of every month
System.schedule(
    'Monthly Placement Report - 1st of Month',
    '0 0 0 1 * ?',
    new MonthlyPlacementReportScheduler()
);
```

**Why `Batch Apex` + `Schedulable`?**
- The data volume (thousands of records) exceeds synchronous and `@future` limits.
- `Database.Batchable` processes records in configurable chunks (default 200), allowing safe traversal of the entire dataset.
- `Database.Stateful` enables accumulating counters (`totalPlaced`, `totalWithdrawn`) across chunks — critical for report generation.
- `Schedulable` enables automated, recurring execution without manual intervention.

---

### Asynchronous Mechanism Selection Guide

| Mechanism | Best For | Key Constraint |
|-----------|----------|---------------|
| `@future` | Simple single callouts or one-off background operations | Cannot be monitored, cannot chain, cannot take sObject arguments |
| `Queueable` | Chainable jobs, complex logic, needs job ID for monitoring | Limited chaining depth (max 5 levels in a single transaction) |
| `Batch Apex` | Processing large datasets (millions of records) | Async — results not immediately available |
| `Schedulable` | Running any job on a time-based schedule | Delegates to `@future`, `Queueable`, or `Batch` for the actual work |

---

## 2. How Do Governor Limits Compare: Async vs. Sync?

Asynchronous Apex runs in its own transaction with significantly relaxed governor limits, allowing operations that would fail in a synchronous context.

### Limit Comparison Table

| Governor Limit | Synchronous | Asynchronous (Future / Queueable) | Batch Apex (per chunk) |
|----------------|-------------|----------------------------------|------------------------|
| **SOQL Queries** | 100 | 200 | 200 |
| **SOQL Query Rows** | 50,000 | 50,000 | 50,000 |
| **DML Statements** | 150 | 150 | 150 |
| **DML Rows** | 10,000 | 10,000 | 10,000 |
| **CPU Time** | 10,000 ms | 60,000 ms | 60,000 ms |
| **Heap Size** | 6 MB | 12 MB | 12 MB |
| **Callouts** | 100 | 100 | 100 (per chunk) |
| **Total Records Processable** | ~10,000 | ~50,000 | Unlimited (via chunking) |

### Key Differences

- **CPU Time** is 6× higher in async (60 seconds vs. 10 seconds), allowing far more complex processing per transaction.
- **Heap Size** doubles to 12 MB in async, enabling larger in-memory data structures.
- **Batch Apex** has no practical limit on total records because it resets the transaction (and all limits) for every chunk — only the per-chunk limits apply.
- **`@ReadOnly` annotation** on a `Batch.start()` method raises the SOQL row limit to **1,000,000** for the query locator phase.

---

## 3. How Did You Handle State, Chaining & Callouts?

### State Management with `Database.Stateful`

By default, Batch Apex re-initializes instance variables between each chunk. Adding `implements Database.Stateful` to the batch class preserves instance variable values across all `execute()` invocations, enabling accumulation patterns like counters and aggregations.

```apex
// Without Database.Stateful: totalPlaced resets to 0 on every chunk
// With Database.Stateful:    totalPlaced accumulates across all chunks
public class MonthlyPlacementReportBatch implements Database.Batchable<sObject>, Database.Stateful {
    private Integer totalPlaced = 0; // Preserved across chunks
}
```

**Trade-off:** `Database.Stateful` increases heap usage because instance state is serialized and passed between chunks. Use it only when accumulation is genuinely needed.

---

### Transaction Chaining with `Queueable`

For the use case where a completed batch needs to trigger a follow-up operation (e.g., send a report email after generating the report), a `Queueable` job was enqueued from the `finish()` method of the batch.

```apex
public void finish(Database.BatchableContext bc) {
    // Insert the summary report record
    insert reportRecord;

    // Chain a Queueable job to send the notification email
    System.enqueueJob(new PlacementReportNotificationJob(reportRecord.Id));
}
```

```apex
public class PlacementReportNotificationJob implements Queueable, Database.AllowsCallouts {
    private Id reportId;

    public PlacementReportNotificationJob(Id reportId) {
        this.reportId = reportId;
    }

    public void execute(QueueableContext ctx) {
        // Send notification email or external API call with report summary
        Messaging.sendEmail(/* ... */);
    }
}
```

**Why Queueable for chaining instead of `@future`?**
- `Queueable` can be enqueued from a `finish()` method, making it the standard "next step" pattern.
- `Queueable` implements `Database.AllowsCallouts`, enabling HTTP requests.
- `Queueable` returns a `Job ID` that can be stored and monitored programmatically.
- `@future` cannot be called from a `finish()` method in all contexts, and cannot be monitored.

---

### Handling External Callouts in Async Context

Synchronous Apex **cannot make HTTP callouts** if the transaction already has pending (uncommitted) DML — a common source of `System.CalloutException: You have uncommitted work pending` errors.

The solution is to move all callouts to `@future(callout=true)` or `Queueable` (with `Database.AllowsCallouts`), ensuring they execute in a fresh transaction where no DML is pending.

```apex
// Trigger fires, DML has occurred — callout is not allowed here
trigger PlacementTrigger on Placement__c (after insert) {
    // This would throw CalloutException:
    // Http http = new Http(); // NOT allowed — DML is pending

    // This is correct — offload the callout to a future method:
    JobBoardIntegrationService.notifyJobBoard(
        Trigger.new[0].Id,
        Trigger.new[0].Status__c
    );
}
```

---

## 4. What Did You Learn Today?

- **Asynchronous Apex is essential for production-grade Salesforce development.** Any operation involving external APIs, large datasets, or time-based scheduling must be handled asynchronously.
- **Choosing the right async mechanism is a design decision.** `@future` for simple callouts, `Queueable` for chaining and monitoring, `Batch` for massive data volumes, and `Schedulable` for time-based automation.
- **`Database.Stateful` is the key to stateful batch processing.** Without it, every chunk starts fresh. With it, you can accumulate totals, track errors, and build summaries across millions of records.
- **`finish()` is the ideal place to chain the next job.** The `finish()` method runs after all chunks complete, making it the natural handoff point for "what happens after the batch is done."
- **Callouts are never allowed inside synchronous DML transactions.** This constraint forces good architecture — keep data operations and external API calls in separate transaction contexts.
- **Cron expressions control scheduled jobs precisely.** `'0 0 0 1 * ?'` means "at 00:00:00, on the 1st day, of every month" — understanding the six-field Salesforce cron format is essential for building reliable scheduled automation.
- **Governor limits in async are more generous, but they still exist.** Async is not unlimited — it simply gives more headroom. Always write bulk-safe code regardless of context.

---

*Bootcamp Status: In Progress*
