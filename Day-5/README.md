# Day 5: SOQL & SOSL Data Retrieval

> **Salesforce Platform Developer Bootcamp — Day 5**

This document covers database querying in Apex using **SOQL (Salesforce Object Query Language)** and **SOSL (Salesforce Object Search Language)** — the two primary mechanisms for retrieving data from the Salesforce database within Apex code.

---

## Table of Contents

1. [SOQL vs. SOSL — Differences & When to Use Each](#1-soql-vs-sosl--differences--when-to-use-each)
2. [Optimizing Queries to Avoid Governor Limits](#2-optimizing-queries-to-avoid-governor-limits)
3. [Parent-to-Child and Child-to-Parent Relationship Queries](#3-parent-to-child-and-child-to-parent-relationship-queries)
4. [What Did You Learn Today?](#4-what-did-you-learn-today)

---

## 1. SOQL vs. SOSL — Differences & When to Use Each

### SOQL — Salesforce Object Query Language

SOQL is modeled after SQL and is used to query **one object at a time** (with related object traversal via relationship queries). It is **precise and structured**, returning a specific list of records that match exact field-level criteria.

```apex
// SOQL: Query a specific set of Placements by Status
List<Placement__c> activePlacements = [
    SELECT Id, Name, Status__c, Candidate__r.Name
    FROM Placement__c
    WHERE Status__c = 'Placed'
    AND CreatedDate = THIS_YEAR
    ORDER BY CreatedDate DESC
    LIMIT 200
];
```

### SOSL — Salesforce Object Search Language

SOSL performs a **full-text search across multiple objects simultaneously** using Salesforce's underlying search index. It is designed for keyword-based search scenarios, similar to a search engine query.

```apex
// SOSL: Search for "Hemanth" across Contacts, Candidates, and Placements
List<List<SObject>> searchResults = [
    FIND 'Hemanth*'
    IN ALL FIELDS
    RETURNING
        Contact(Id, FirstName, LastName, Email),
        Candidate__c(Id, Name, Email__c),
        Placement__c(Id, Name, Status__c)
];
```

### When Was Each Used?

| Scenario | Tool Used | Reasoning |
|----------|-----------|-----------|
| Fetching all Placements for a specific Recruiter by ID | **SOQL** | Exact ID match — SOQL is deterministic and faster for known-field lookups |
| Fetching related Account and Contact data for a Placement | **SOQL** | Relationship traversal is a SOQL strength |
| Building a global "Search Candidates" feature where users type a name | **SOSL** | Full-text search across multiple objects with partial keyword matching |
| Aggregating placement counts per Job Opening | **SOQL with Aggregate Functions** | `COUNT()`, `SUM()`, `GROUP BY` are SOQL capabilities |
| Finding records where any field contains a phone number fragment | **SOSL** | SOSL searches all indexed fields simultaneously |

### Key Differences at a Glance

| Attribute | SOQL | SOSL |
|-----------|------|------|
| Objects queried | One (with traversal) | Multiple simultaneously |
| Search type | Exact field-value matching | Full-text keyword search |
| Returns | `List<SObject>` | `List<List<SObject>>` |
| Governor limit | 100 queries per transaction | 20 searches per transaction |
| Use case | Known field conditions | Keyword search across objects |
| Null results | Empty list `[]` | Empty inner lists |

---

## 2. Optimizing Queries to Avoid Governor Limits

The Salesforce Governor Limit for SOQL is **100 queries per synchronous transaction**. The most common cause of hitting this limit is placing queries inside loops.

### Anti-Pattern: SOQL Inside a Loop

```apex
// BAD — This issues one SOQL query per Placement record
for (Placement__c p : placements) {
    Candidate__c candidate = [SELECT Id, Name FROM Candidate__c WHERE Id = :p.Candidate__c];
    // This hits the 101-query limit on the 101st record
}
```

### Optimized Pattern: Collect → Query → Map → Process

The correct pattern collects all required IDs first, issues a **single bulk query**, stores results in a `Map`, then processes each record using the map for constant-time lookups.

```apex
// GOOD — Single SOQL query for all records

// Step 1: Collect all Candidate IDs from the Placement list
Set<Id> candidateIds = new Set<Id>();
for (Placement__c p : placements) {
    if (p.Candidate__c != null) {
        candidateIds.add(p.Candidate__c);
    }
}

// Step 2: Fetch all Candidates in one query
Map<Id, Candidate__c> candidateMap = new Map<Id, Candidate__c>(
    [SELECT Id, Name, Email__c, Skills__c FROM Candidate__c WHERE Id IN :candidateIds]
);

// Step 3: Process each Placement using the map (zero additional queries)
for (Placement__c p : placements) {
    Candidate__c c = candidateMap.get(p.Candidate__c);
    if (c != null) {
        // Use candidate data safely
    }
}
```

### Additional Optimization Techniques

| Technique | Description |
|-----------|-------------|
| **SELECT only needed fields** | Avoid `SELECT *` equivalents — query only the fields you will actually use to reduce heap consumption |
| **Use `LIMIT`** | Always add `LIMIT` to user-facing queries to cap result sets |
| **Use `WHERE` filters** | Never query all records of an object; always filter to the smallest relevant dataset |
| **Use `@ReadOnly` in Batch Apex** | Raises the query row limit from 50,000 to 1,000,000 for read-heavy batch jobs |
| **Avoid nested queries unnecessarily** | Use relationship queries instead of separate SOQL calls for parent/child data |

---

## 3. Parent-to-Child and Child-to-Parent Relationship Queries

Salesforce relationship queries allow traversal of object relationships without issuing additional SOQL queries, combining related data in a single efficient call.

### Child-to-Parent (Upward Traversal)

Access **parent record fields** from a child record using dot notation on the relationship field name. The parent object name is the **relationship field name** (the lookup/master-detail field) followed by `.FieldName`.

```apex
// Query Placements and access their parent Candidate and Job Opening data
List<Placement__c> placements = [
    SELECT
        Id,
        Name,
        Status__c,
        Candidate__r.Name,         // Parent Candidate's Name
        Candidate__r.Email__c,     // Parent Candidate's Email
        Job_Opening__r.Title__c,   // Parent Job Opening's Title
        Job_Opening__r.Department__c
    FROM Placement__c
    WHERE Status__c IN ('Placed', 'Offer Extended')
];

for (Placement__c p : placements) {
    System.debug('Candidate: ' + p.Candidate__r.Name + ' | Role: ' + p.Job_Opening__r.Title__c);
}
```

**When used:** Whenever child records need context from their parent — e.g., displaying the Candidate's name or email on a Placement card without a separate query.

### Parent-to-Child (Downward Traversal / Sub-query)

Access **child records** from a parent record using a nested sub-query. The child relationship name is typically the **plural API name** of the child object.

```apex
// Query Job Openings and their related child Placement records
List<Job_Opening__c> jobOpenings = [
    SELECT
        Id,
        Title__c,
        Department__c,
        (
            SELECT Id, Name, Status__c, Candidate__r.Name
            FROM Placements__r    // Child relationship name (plural)
            WHERE Status__c != 'Withdrawn'
        )
    FROM Job_Opening__c
    WHERE IsActive__c = true
];

for (Job_Opening__c job : jobOpenings) {
    System.debug('Job: ' + job.Title__c + ' | Applications: ' + job.Placements__r.size());
    for (Placement__c p : job.Placements__r) {
        System.debug('  - ' + p.Candidate__r.Name + ' (' + p.Status__c + ')');
    }
}
```

**When used:** When a parent record needs a summary of all its children — e.g., displaying all candidates who applied to a specific job opening in a single Apex query.

---

## 4. What Did You Learn Today?

- **SOQL is for precision; SOSL is for discovery.** When you know exactly which object and which fields to filter on, use SOQL. When you need to find records containing a keyword across your entire org, use SOSL.
- **`List<List<SObject>>` is SOSL's return type.** Each inner list corresponds to one of the objects in the `RETURNING` clause, in order. Always access them by index: `searchResults[0]` for the first object.
- **Relationship query names matter.** The child relationship name in a parent-to-child query is not always obvious — it is defined in the object's relationship settings and is typically the plural form of the child object's API name (e.g., `Placements__r` for `Placement__c`).
- **Dot notation traversal is free.** Accessing `Candidate__r.Name` in a query result does not issue an additional SOQL query — the data was fetched in the original query. This is a major performance advantage.
- **`WHERE Id IN :collection` is the bulkification workhorse.** This bind variable pattern replaces N queries with 1, and is the foundation of every well-written Apex class.
- **Query plan matters at scale.** Indexed fields (like `Id`, `Name`, custom external ID fields) in `WHERE` clauses lead to faster queries. Unindexed fields on large objects can cause timeout issues in production.

---

*Bootcamp Status: In Progress | Next: Day 6 — Advanced Apex Triggers & Bulkification*
