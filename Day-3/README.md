# Day 3: Validation Rules, Flows & Triggers

> **Salesforce Platform Developer Bootcamp — Day 3**

This document details the architectural decisions made when building the **Placement Management System** automation layer. Each business requirement was deliberately matched to the most appropriate Salesforce automation tool: Flow, Validation Rules, or Apex Trigger.

---

## Table of Contents

1. [Which Requirements Did You Solve Using Flow?](#1-which-requirements-did-you-solve-using-flow)
2. [Which Requirements Required Validation Rules?](#2-which-requirements-required-validation-rules)
3. [Which Requirements Still Needed Apex?](#3-which-requirements-still-needed-apex)
4. [Why Did You Choose Those Solutions?](#4-why-did-you-choose-those-solutions)
5. [What Did You Learn Today?](#5-what-did-you-learn-today)

---

## 1. Which Requirements Did You Solve Using Flow?

**Record-Triggered Flows** were used for automation that could be expressed declaratively without code, particularly for straightforward field updates and notifications triggered by record state changes.

### Requirements Handled by Flow

| Requirement | Flow Type | Trigger Condition |
|-------------|-----------|-------------------|
| Auto-assign a Recruitment Manager to a new Placement record | Record-Triggered Flow | When a Placement record is created |
| Send an email notification to the candidate when Placement status changes to "Offer Extended" | Record-Triggered Flow | When `Status__c` changes to `Offer Extended` |
| Create a follow-up Task 30 days after a Placement is marked "Placed" | Scheduled-Triggered Flow | Runs daily, filters on `Status__c = 'Placed'` |
| Update the parent Job Opening's `Filled__c` checkbox when Placement is marked "Placed" | Record-Triggered Flow | After-save, when `Status__c` changes to `Placed` |

**Why Flow was the right choice here:** All of these requirements follow a clear "if record state = X, then do Y" pattern — ideal for declarative tools. No cross-object loops, no complex data aggregations, and no Governor Limit risks that require Apex-level control.

---

## 2. Which Requirements Required Validation Rules?

**Validation Rules** were used to enforce data integrity constraints at the point of record save, ensuring that invalid data is rejected before it persists to the database.

### Requirements Handled by Validation Rules

| Requirement | Validation Rule Formula | Error Message |
|-------------|------------------------|---------------|
| Placement Start Date must not be in the past | `StartDate__c < TODAY()` | "Start Date cannot be a past date." |
| Offer Salary is required when Status is "Offer Extended" | `ISPICKVAL(Status__c, 'Offer Extended') && ISBLANK(OfferSalary__c)` | "Offer Salary is required when extending an offer." |
| A Placement cannot move back to "Applied" from a later stage | `ISPICKVAL(Status__c, 'Applied') && NOT(ISNEW())` | "You cannot revert a Placement back to the Applied stage." |
| Candidate Email must follow a valid format | `NOT(REGEX(Email__c, '[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\\.[a-zA-Z]{2,}'))` | "Please enter a valid email address." |

**Why Validation Rules were the right choice here:** These are pure **data quality constraints** — binary pass/fail checks against field values. Validation Rules fire consistently across all entry points (UI, API, data loader) and are the most efficient mechanism for this purpose, requiring zero code and zero execution overhead.

---

## 3. Which Requirements Still Needed Apex?

**Apex Triggers** were reserved for requirements that exceeded the capabilities of declarative tools — specifically those involving complex logic, aggregation across many records, or precise transactional control.

### Requirements Handled by Apex

| Requirement | Reason Apex Was Needed |
|-------------|------------------------|
| Auto-calculate and populate `DaysToPlace__c` (days between Application Date and Placement Date) across bulk imports | Flow cannot reliably handle bulk record processing (200+ records) without hitting limits. Apex with bulkified trigger handles this safely. |
| Roll up the count of active Placements per Recruiter onto the `Recruiter__c` object | Standard Roll-Up Summary fields do not work across lookup relationships. Apex trigger maintains the count with a single aggregated query. |
| Prevent duplicate Placement records for the same Candidate + Job Opening combination | Duplicate detection requires a SOQL query against existing records, which is only possible in Apex. |
| Log a `Placement_Audit__c` record whenever a Placement status changes, capturing old and new values | Requires access to `Trigger.oldMap` to compare field values — only available in Apex triggers. |

---

## 4. Why Did You Choose Those Solutions?

The selection framework followed Salesforce's **"Declarative First, Code Second"** principle:

```
Business Requirement
        |
        v
Can it be solved with a Validation Rule?
  YES --> Use Validation Rule (fastest, zero maintenance)
  NO  --> Can it be solved with Flow?
            YES --> Use Flow (visual, declarative, auditable)
            NO  --> Use Apex Trigger (full programmatic control)
```

### Decision Rationale Summary

| Tool | Strengths | Used When |
|------|-----------|-----------|
| **Validation Rule** | Zero code, fires on all entry points, immediate feedback | Enforcing field-level data integrity constraints |
| **Flow** | Visual logic, no code, supports automation and notifications | Straightforward "if this, then that" record automation |
| **Apex Trigger** | Full programmatic power, bulk-safe, access to all context variables | Complex logic, cross-object aggregation, duplicate detection, audit logging |

This layered architecture keeps the system **maintainable** — admins can manage Flows and Validation Rules without developer involvement, while Apex handles the complex edge cases that require code review and testing.

---

## 5. What Did You Learn Today?

- **Tool selection matters more than code quality.** A perfectly written Apex trigger is still the wrong solution if a Validation Rule or Flow could have done the job declaratively.
- **Validation Rules are underrated.** They enforce data quality at the platform level, firing consistently regardless of how data enters the system (UI, API, Data Loader, Trigger DML).
- **Flows have limits.** Record-Triggered Flows are powerful for single-record scenarios but can behave unpredictably during bulk operations. Always consider Governor Limits before choosing Flow over Apex for data-intensive processes.
- **`Trigger.oldMap` is essential for change detection.** Comparing `Trigger.new` values against `Trigger.oldMap` is the standard pattern for detecting what actually changed on a record update — something Flow cannot replicate with the same precision.
- **Layered automation is a sign of good architecture.** The best Salesforce solutions combine all three tools strategically rather than forcing every requirement through a single mechanism.

---

*Bootcamp Status: In Progress | Next: Day 4 — Your First Lightning Web Component (LWC)*
