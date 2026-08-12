# MediCare Hospital Management System

## Salesforce Developer Project

A practical Hospital Management System built on the Salesforce Platform to learn and demonstrate Salesforce Administration, Apex Development, SOQL, DML, Triggers, Flows, Lightning Web Components, Security, Testing, Debugging, and Deployment.

---

# 1. Project Overview

The **MediCare Hospital Management System** is a Salesforce-based application designed to manage the core operations of a hospital.

The project manages:

* Patients
* Doctors
* Appointments
* Medical Records
* Prescriptions
* Bills
* Hospital reporting
* Dashboard information
* Business automation
* User access and security

The primary purpose of this project is not only to create a working hospital application, but also to provide a practical environment for learning Salesforce development concepts from beginner level through intermediate/advanced developer concepts.

The project starts with a simple data model and progressively introduces Salesforce technologies.

---

# 2. Project Objectives

The main objectives are:

1. Build a functional Hospital Management System using Salesforce.
2. Learn Salesforce development through a real-world project.
3. Understand Salesforce's declarative and programmatic development models.
4. Practice Apex with real Salesforce records.
5. Learn SOQL and DML through actual business requirements.
6. Understand Salesforce Governor Limits.
7. Build bulkified Apex code.
8. Implement Trigger Handler architecture.
9. Use Flow for declarative automation.
10. Build Lightning Web Components.
11. Implement Salesforce security.
12. Create Apex test classes.
13. Practice debugging and error handling.
14. Practice Salesforce CLI and metadata deployment.
15. Build a project that can be discussed in Salesforce Developer interviews.

---

# 3. Technology Stack

## Salesforce Platform

* Salesforce Developer Org
* Lightning Experience
* Lightning App
* Custom Objects
* Custom Fields
* Lookup Relationships
* Record-Triggered Flows
* Validation Rules
* Page Layouts
* Reports
* Dashboards
* Profiles
* Permission Sets
* Roles
* Sharing

## Development

* Apex
* SOQL
* SOSL
* DML
* Apex Collections
* Apex Classes
* Triggers
* Trigger Handler Pattern
* Exception Handling
* Test Classes
* Lightning Web Components

## Development Tools

* Visual Studio Code
* Salesforce CLI
* Salesforce Extensions for VS Code
* Git
* GitHub

---

# 4. High-Level Architecture

The system follows a layered Salesforce architecture.

```text
                    MediCare Hospital
                           |
        +------------------+------------------+
        |                  |                  |
     Patients           Doctors          Appointments
        |                  |                  |
        +------------------+------------------+
                           |
                    Medical Records
                           |
                    Prescriptions
                           |
                         Bills
```

The technical architecture is:

```text
                    Lightning UI
                         |
              +----------+----------+
              |                     |
       Standard Salesforce       LWC
             UI                    |
              |                     |
              +----------+----------+
                         |
                    Apex Classes
                         |
                  Business Logic
                         |
                    SOQL / DML
                         |
                  Salesforce DB
```

Automation:

```text
Record Change
     |
     +---- Flow
     |
     +---- Trigger
             |
        Trigger Handler
             |
        Service Layer
             |
          SOQL/DML
```

---

# 5. Data Model

The initial data model contains six main objects.

```text
Patient
Doctor
Appointment
Medical Record
Prescription
Bill
```

---

# 6. Patient Object

API Name:

```text
Patient__c
```

The Patient object stores information about people receiving treatment at the hospital.

Typical fields include:

* Patient Name
* Blood Group
* Phone
* Email
* Date of Birth
* Gender
* Address
* Status

Example:

```text
Patient Name: Ravi Kumar
Blood Group: O+
Phone: 9876543210
Status: Active
```

---

# 7. Doctor Object

API Name:

```text
Doctor__c
```

The Doctor object stores hospital doctor information.

Typical fields include:

* Doctor Name
* Specialization
* Phone
* Email
* Availability
* Status

Example:

```text
Doctor Name: Dr. Suresh
Specialization: Cardiology
Status: Active
```

---

# 8. Appointment Object

API Name:

```text
Appointment__c
```

The Appointment object connects a patient with a doctor.

Typical fields include:

* Appointment Name
* Patient
* Doctor
* Appointment Date
* Appointment Time
* Status
* Reason/Description

Example:

```text
Patient: Ravi Kumar
Doctor: Dr. Suresh
Date: 12-Aug-2026
Status: Scheduled
```

---

# 9. Medical Record Object

API Name:

```text
Medical_Record__c
```

The Medical Record object stores information about a patient's treatment history.

Possible fields:

* Patient
* Doctor
* Appointment
* Diagnosis
* Medical Notes
* Treatment
* Record Date

Example:

```text
Patient: Ravi Kumar
Doctor: Dr. Suresh
Diagnosis: Hypertension
Treatment: Medication + Follow-up
```

---

# 10. Prescription Object

API Name:

```text
Prescription__c
```

The Prescription object stores medicines prescribed by doctors.

Possible fields:

* Patient
* Doctor
* Appointment
* Medical Record
* Medicine
* Dosage
* Frequency
* Duration
* Instructions

Example:

```text
Patient: Ravi Kumar
Medicine: Medicine A
Dosage: 500mg
Frequency: Twice Daily
Duration: 5 Days
```

---

# 11. Bill Object

API Name:

```text
Bill__c
```

The Bill object manages hospital billing information.

Possible fields:

* Patient
* Appointment
* Amount
* Payment Status
* Billing Date
* Description

Example:

```text
Patient: Ravi Kumar
Amount: ₹2500
Payment Status: Pending
```

---

# 12. Object Relationships

The core relationship is:

```text
Patient
   |
   | 1
   |
   |----< Appointment >----|
                            |
                            | 1
                          Doctor
```

One Patient can have many Appointments.

One Doctor can have many Appointments.

Each Appointment belongs to one Patient and one Doctor.

This is implemented using Lookup Relationships.

---

# 13. Salesforce Relationship Querying

Because Appointment contains lookups to Patient and Doctor, relationship fields can be accessed using `__r`.

Example:

```apex
SELECT Id,
       Name,
       Patient__r.Name,
       Doctor__r.Name
FROM Appointment__c
```

Conceptually:

```text
Appointment
    |
    +-- Patient__r.Name
    |
    +-- Doctor__r.Name
```

This allows the application to display related record information without duplicating the data.

---

# 14. Lightning App

A Salesforce Lightning App is used as the main navigation interface.

The application provides access to:

* Patients
* Doctors
* Appointments
* Medical Records
* Prescriptions
* Bills
* Reports
* Dashboards

VS Code connects to the Salesforce Org, not separately to the Lightning App.

Architecture:

```text
VS Code
   |
   v
Salesforce Org
   |
   +--- Lightning App
   +--- Objects
   +--- Apex
   +--- Flow
   +--- LWC
```

---

# 15. Salesforce Development Layers

The project progressively demonstrates the following layers.

## Data Layer

* Custom Objects
* Fields
* Relationships
* Records

## Query Layer

* SOQL
* SOSL

## Business Logic Layer

* Apex Classes
* Collections
* DML
* Exception Handling

## Automation Layer

* Flows
* Triggers
* Trigger Handlers

## Presentation Layer

* Lightning Experience
* Lightning App
* LWC

## Security Layer

* Profiles
* Permission Sets
* Roles
* Sharing
* Object permissions
* Field-level security

## Quality Layer

* Test Classes
* Debug Logs
* Error Handling
* Governor Limit analysis

## Deployment Layer

* Salesforce CLI
* Metadata
* Git
* Deployment

---

# 16. Apex Development

Apex is used for complex server-side business logic.

The project uses Apex for:

* Querying records
* Processing records
* Creating records
* Updating records
* Deleting records
* Complex validation
* Business logic
* Trigger processing
* LWC backend services

---

# 17. Apex Collections

The project demonstrates:

## List

Used for ordered collections.

```apex
List<Patient__c> patients;
```

## Set

Used for unique values.

```apex
Set<Id> patientIds;
```

## Map

Used for key-value relationships.

```apex
Map<Id, Patient__c> patientMap;
```

A common Salesforce bulkification pattern is:

```text
Trigger Records
      |
      v
Collect IDs into Set
      |
      v
Execute one SOQL query
      |
      v
Store results in Map
      |
      v
Process records
```

---

# 18. SOQL

SOQL is used to query Salesforce data.

Examples:

```apex
SELECT Id, Name
FROM Patient__c
```

Filtering:

```apex
SELECT Id, Name
FROM Patient__c
WHERE Name LIKE 'R%'
```

Ordering:

```apex
SELECT Id, Name
FROM Patient__c
ORDER BY Name
```

Limiting:

```apex
SELECT Id, Name
FROM Patient__c
LIMIT 10
```

Relationship query:

```apex
SELECT Id,
       Name,
       Patient__r.Name,
       Doctor__r.Name
FROM Appointment__c
```

---

# 19. DML

The project will demonstrate Salesforce DML operations.

```text
insert
update
delete
undelete
upsert
```

Example:

```apex
Patient__c patient = new Patient__c(
    Name = 'Test Patient'
);

insert patient;
```

DML is used whenever Apex needs to modify Salesforce records.

---

# 20. Apex Service Classes

The project uses service classes to keep business logic organized.

Planned classes:

```text
PatientService.cls
DoctorService.cls
AppointmentService.cls
HospitalDashboardService.cls
HospitalException.cls
AppointmentTriggerHandler.cls
```

The goal is to avoid putting large amounts of business logic directly inside triggers or LWC components.

---

# 21. PatientService

Purpose:

Handle reusable Patient-related operations.

Responsibilities:

* Retrieve patients
* Retrieve individual patients
* Search/filter patients
* Perform patient-related business operations

Example structure:

```apex
public with sharing class PatientService {
    public static List<Patient__c> getPatients() {
        ...
    }
}
```

---

# 22. DoctorService

Purpose:

Handle reusable Doctor-related operations.

Responsibilities:

* Retrieve doctors
* Retrieve individual doctors
* Filter doctors
* Doctor-related business logic

---

# 23. AppointmentService

Purpose:

Handle Appointment-related operations.

Responsibilities:

* Retrieve appointments
* Retrieve appointments for a patient
* Retrieve appointments for a doctor
* Appointment-related business logic

---

# 24. HospitalDashboardService

Purpose:

Provide aggregated data for dashboards and Lightning Web Components.

Possible statistics:

* Total Patients
* Total Doctors
* Total Appointments
* Pending Appointments
* Completed Appointments
* Pending Bills
* Revenue

---

# 25. Exception Handling

The project will demonstrate:

```text
Exception
   |
   +-- QueryException
   +-- DmlException
   +-- NullPointerException
   +-- ListException
   +-- Custom Exception
```

A custom exception is included:

```apex
HospitalException
```

The objective is to understand how Salesforce errors occur and how they should be handled.

---

# 26. Trigger Architecture

Triggers should remain lightweight.

Target architecture:

```text
AppointmentTrigger
       |
       v
AppointmentTriggerHandler
       |
       v
Service / Business Logic
       |
       v
SOQL / DML
```

The trigger should not contain large amounts of business logic.

---

# 27. Trigger Contexts

The project will demonstrate:

```text
before insert
after insert
before update
after update
before delete
after delete
after undelete
```

Important context variables:

```text
Trigger.new
Trigger.old
Trigger.newMap
Trigger.oldMap
Trigger.isInsert
Trigger.isUpdate
Trigger.isDelete
Trigger.isBefore
Trigger.isAfter
```

---

# 28. Bulkification

All trigger logic must be designed to handle multiple records.

The system should work correctly for:

```text
1 record
10 records
100 records
200 records
```

Bad approach:

```apex
for (Appointment__c appointment : Trigger.new) {
    // SOQL
}
```

Good approach:

```text
Collect IDs
    ↓
One SOQL query
    ↓
Map records
    ↓
Process records
    ↓
One DML operation where possible
```

---

# 29. Governor Limits

Salesforce is a multitenant platform and enforces governor limits.

The project will teach:

* SOQL query limits
* Query row limits
* DML statement limits
* DML row limits
* CPU time
* Heap size
* Callout limits
* Async Apex limits
* Email limits

Debug logs will be used to understand resource consumption.

Example:

```text
Number of SOQL queries: X out of 100
Number of DML statements: X out of 150
Number of query rows: X out of 50000
Maximum CPU time: X out of 10000
Maximum heap size: X out of 6000000
```

---

# 30. Flow Automation

Flow will be used for declarative automation.

Possible hospital flows:

## Appointment Flow

When an Appointment is created:

```text
Appointment Created
        |
        v
Validate Status
        |
        v
Perform required automation
```

## Appointment Completion Flow

When an Appointment becomes Completed:

```text
Status = Completed
        |
        v
Create/Update Medical Record
```

## Billing Flow

When an appointment is completed:

```text
Appointment Completed
        |
        v
Create Bill
```

Flow will be preferred when the automation can be implemented cleanly without Apex.

---

# 31. Validation Rules

Validation rules prevent invalid hospital data.

Examples:

* Appointment cannot be created without Patient.
* Appointment cannot be created without Doctor.
* Appointment date cannot be invalid.
* Bill amount cannot be negative.
* Required patient information must be present.
* Invalid status transitions should be prevented where appropriate.

The project will demonstrate the difference between:

```text
Validation Rule
vs
Flow
vs
Apex
vs
Trigger
```

---

# 32. Page Layouts

Page layouts will control:

* Field visibility
* Required fields
* Sections
* Related Lists
* Record page organization

Different layouts can eventually be provided for different users where required.

---

# 33. Lightning App Builder

Lightning App Builder will be used to create useful record pages and dashboards.

Possible components:

* Patient information
* Appointment information
* Related appointments
* Medical records
* Prescriptions
* Billing
* Custom LWC components

---

# 34. Reports

The project will include reports such as:

## Patient Report

Displays:

* Patient
* Blood Group
* Status
* Contact information

## Appointment Report

Displays:

* Patient
* Doctor
* Date
* Status

## Doctor Workload Report

Displays:

* Doctor
* Number of appointments
* Completed appointments
* Pending appointments

## Billing Report

Displays:

* Patient
* Bill Amount
* Payment Status

---

# 35. Dashboard

The hospital dashboard will provide a high-level view.

Possible dashboard metrics:

```text
Total Patients
Total Doctors
Total Appointments
Scheduled Appointments
Completed Appointments
Cancelled Appointments
Pending Bills
Total Revenue
```

Possible visualizations:

* Appointments by Status
* Appointments by Doctor
* Patients by Blood Group
* Revenue by Month
* Appointment Trends

---

# 36. Lightning Web Components

LWC will provide custom user interfaces.

Planned components:

```text
hospitalDashboard
patientSearch
patientDetails
appointmentManager
doctorAppointments
patientAppointmentHistory
```

---

# 37. LWC + Apex Architecture

The intended architecture is:

```text
Lightning Web Component
          |
          v
      Apex Method
          |
          v
    Service Class
          |
          v
      SOQL / DML
          |
          v
 Salesforce Database
```

This demonstrates how modern Salesforce applications communicate between frontend and backend.

---

# 38. LWC Features

The project will demonstrate:

* HTML templates
* JavaScript
* CSS
* `@wire`
* Imperative Apex
* Apex imports
* Events
* Parent-child communication
* Conditional rendering
* Iteration
* Forms
* Lightning Data Table
* Loading indicators
* Error handling
* Refreshing data

---

# 39. Security

The project will demonstrate Salesforce security concepts.

## Object-Level Security

Controls access to:

* Patient
* Doctor
* Appointment
* Medical Record
* Prescription
* Bill

## Field-Level Security

Controls access to sensitive fields.

## Profiles

Define baseline permissions.

## Permission Sets

Provide additional permissions without creating unnecessary profiles.

## Roles

Control record visibility through hierarchy.

## Sharing

Controls access to records.

---

# 40. Apex Security

The project will also demonstrate:

```apex
with sharing
```

and appropriate security considerations around:

* CRUD
* FLS
* Record sharing
* User permissions

Apex should not unintentionally expose data a user should not access.

---

# 41. Test Classes

All important Apex logic will eventually have test classes.

Testing will include:

* Positive scenarios
* Negative scenarios
* Bulk scenarios
* Trigger scenarios
* Exception scenarios
* Boundary cases

Important testing tools:

```apex
@testSetup
Test.startTest()
Test.stopTest()
System.assert()
System.assertEquals()
System.assertNotEquals()
```

The goal is not simply to achieve code coverage.

The goal is to verify that the application behaves correctly.

---

# 42. Debugging Strategy

Debugging is a major learning objective of this project.

The project intentionally uses debugging exercises.

Typical debugging workflow:

```text
Error
  |
  v
Read error message
  |
  v
Find class/trigger/component
  |
  v
Find line number
  |
  v
Identify root cause
  |
  v
Reproduce
  |
  v
Fix
  |
  v
Retest
```

Important debugging tools:

* Apex Debug Logs
* `System.debug()`
* Developer Console
* VS Code
* Salesforce CLI
* Browser Developer Tools for LWC

---

# 43. Common Errors to Practice

The project will deliberately expose realistic errors.

## Apex

* NullPointerException
* ListException
* Invalid conversion
* Incorrect variable type
* Incorrect method signature
* Invalid access modifier

## SOQL

* Invalid field
* Invalid relationship
* Invalid object
* Query syntax error
* No records returned
* Too many records

## DML

* DmlException
* Required field missing
* Validation Rule failure
* Duplicate data
* Invalid lookup
* Permission/security failure

## Triggers

* SOQL inside loop
* DML inside loop
* Governor limit exceeded
* Recursion
* Incorrect trigger context

## LWC

* Invalid Apex import
* Apex method errors
* Wire errors
* JavaScript errors
* Template errors
* Incorrect property references

## Deployment

* Compilation errors
* Missing metadata
* Test failures
* Incorrect API names
* Dependency problems

---

# 44. Salesforce CLI

The project is developed using Salesforce CLI and VS Code.

Common operations include:

```text
Authorize Org
Create Project
Create Apex Class
Create Trigger
Deploy Source
Retrieve Source
Run Anonymous Apex
Run Apex Tests
View Debug Logs
```

The project will gradually use source-driven development.

---

# 45. Source Structure

Current/target project structure:

```text
force-app/
└── main/
    └── default/
        ├── classes/
        │   ├── PatientService.cls
        │   ├── DoctorService.cls
        │   ├── AppointmentService.cls
        │   ├── HospitalDashboardService.cls
        │   ├── HospitalException.cls
        │   └── AppointmentTriggerHandler.cls
        │
        ├── triggers/
        │   └── AppointmentTrigger.trigger
        │
        ├── objects/
        │   ├── Patient__c/
        │   ├── Doctor__c/
        │   ├── Appointment__c/
        │   ├── Medical_Record__c/
        │   ├── Prescription__c/
        │   └── Bill__c/
        │
        ├── lwc/
        │
        ├── flows/
        │
        ├── layouts/
        │
        ├── permissionsets/
        │
        ├── profiles/
        │
        └── reports/
```

---

# 46. Development Phases

The project is developed progressively.

## Phase 1 — Salesforce Foundation

* Org setup
* Objects
* Fields
* Relationships
* Records
* Lightning App

## Phase 2 — SOQL

* Basic queries
* Filtering
* Sorting
* Limits
* Relationship queries
* Parent-to-child
* Child-to-parent

## Phase 3 — Apex Fundamentals

* Variables
* Data types
* Operators
* Conditions
* Loops
* Methods
* Debugging

## Phase 4 — Collections

* List
* Set
* Map
* SObject collections
* Bulk processing concepts

## Phase 5 — DML

* Insert
* Update
* Delete
* Undelete
* Upsert
* DML exception handling

## Phase 6 — Apex Classes

* Classes
* Methods
* Parameters
* Return types
* Static methods
* Access modifiers
* Service layer

## Phase 7 — Triggers

* Trigger contexts
* Trigger.new
* Trigger.old
* Trigger.newMap
* Trigger.oldMap
* Before/after events

## Phase 8 — Trigger Handler

* Separation of concerns
* Handler classes
* Service classes
* Bulkification

## Phase 9 — Governor Limits

* SOQL limits
* DML limits
* CPU
* Heap
* Query rows
* Bulk processing

## Phase 10 — Flow

* Record-triggered Flow
* Decisions
* Get Records
* Create Records
* Update Records
* Fault handling

## Phase 11 — UI

* Page Layouts
* Lightning App Builder
* Record Pages
* Related Lists

## Phase 12 — LWC

* Components
* Apex integration
* Wire
* Imperative Apex
* Events
* Datatables
* Forms
* Error handling

## Phase 13 — Security

* Profiles
* Permission Sets
* Roles
* Sharing
* CRUD/FLS

## Phase 14 — Testing

* Test classes
* Test data
* Assertions
* Bulk testing
* Negative testing
* Trigger testing

## Phase 15 — Deployment

* Salesforce CLI
* Source tracking
* Metadata
* Git
* Deployment
* Troubleshooting deployment failures

---

# 47. Current Development Status

Completed:

* Salesforce Org
* Salesforce DX Project
* Org Authentication
* Custom Objects
* Fields
* Relationships
* Records
* Lightning App
* SOQL
* Relationship Queries
* Anonymous Apex
* Apex Variables
* Apex Data Types
* Apex Debug Logs
* Basic Apex + SOQL

Currently learning:

```text
Apex Collections
    |
    +-- List
    +-- Set
    +-- Map
```

Next:

```text
DML
```

Then:

```text
Apex Classes
```

Then:

```text
Triggers
```

---

# 48. Development Principles

The project follows these principles:

1. Keep triggers lightweight.
2. Avoid SOQL inside loops.
3. Avoid DML inside loops.
4. Bulkify Apex.
5. Reuse service classes.
6. Keep business logic separated.
7. Validate data at the appropriate Salesforce layer.
8. Use Flow when declarative automation is sufficient.
9. Use Apex when logic is complex or requires programmatic control.
10. Respect governor limits.
11. Handle exceptions properly.
12. Write meaningful tests.
13. Do not hardcode IDs.
14. Do not guess Salesforce API names.
15. Follow Salesforce naming conventions.
16. Keep code readable.
17. Prefer maintainability over unnecessary complexity.
18. Debug using evidence from logs and errors.

---

# 49. Future Enhancements

After completing the core system, possible future enhancements include:

* Patient self-service portal
* Doctor availability management
* Appointment conflict detection
* Email notifications
* SMS integration
* Calendar integration
* Advanced billing
* Payment gateway integration
* External healthcare API integration
* Platform Events
* Queueable Apex
* Batch Apex
* Scheduled Apex
* Named Credentials
* REST API integration
* External system integration

These are optional and should only be added after the core Salesforce developer concepts are mastered.

---

# 50. Final Project Goal

The final MediCare Hospital Management System should demonstrate the following Salesforce Developer skill set:

```text
                 MEDICARE HMS
                      |
        +-------------+-------------+
        |                           |
   Declarative                 Programmatic
        |                           |
 Objects                      Apex
 Fields                       SOQL
 Relationships                DML
 Page Layouts                 Classes
 Validation Rules             Triggers
 Flow                         Collections
 Reports                      Bulkification
 Dashboards                   Governor Limits
 Lightning App                Test Classes
        |                           |
        +-------------+-------------+
                      |
                     LWC
                      |
                   Security
                      |
                  Deployment
```

The project should be simple enough to understand but comprehensive enough to demonstrate real Salesforce development practices.

The ultimate objective is not just:

> "I built a Hospital Management System."

The objective is:

> "I understand how Salesforce applications are designed, configured, developed, automated, secured, tested, debugged, and deployed."

---

# 51. Learning Objective

By the end of this project, I should be able to:

* Understand an existing Salesforce Org.
* Design a basic Salesforce data model.
* Create and relate custom objects.
* Write efficient SOQL.
* Write Apex.
* Use List, Set, and Map.
* Perform DML.
* Build reusable Apex classes.
* Build bulkified triggers.
* Implement Trigger Handler architecture.
* Understand Governor Limits.
* Build Record-Triggered Flows.
* Create validation rules.
* Configure Lightning pages.
* Build reports and dashboards.
* Develop LWC components.
* Connect LWC with Apex.
* Apply Salesforce security.
* Write Apex test classes.
* Read Salesforce debug logs.
* Diagnose common Salesforce errors.
* Use Salesforce CLI.
* Deploy Salesforce metadata.
* Explain the architecture in a Salesforce Developer interview.

---

# 52. Project Status

**Project:** MediCare Hospital Management System

**Platform:** Salesforce

**Development Environment:** VS Code + Salesforce CLI

**Current Stage:** Apex Collections

**Next Stage:** DML

**Final Goal:** Full-stack Salesforce Hospital Management System demonstrating Salesforce Administration + Development + LWC + Automation + Security + Testing + Deployment.

---

## Author

**Hemanth**

Final-year B.Tech Computer Science student

Learning and developing Salesforce through the MediCare Hospital Management System.
