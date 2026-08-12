# Day 4: Your First Lightning Web Component (LWC)

> **Salesforce Platform Developer Bootcamp — Day 4**

This document covers the introduction to frontend development on the Salesforce platform using **Lightning Web Components (LWC)** — Salesforce's modern, standards-based framework for building user interfaces.

---

## Table of Contents

1. [What Is LWC?](#1-what-is-lwc)
2. [What Did You Build?](#2-what-did-you-build)
3. [Which File Contains HTML?](#3-which-file-contains-html)
4. [Which File Contains JavaScript?](#4-which-file-contains-javascript)
5. [What Did You Learn Today?](#5-what-did-you-learn-today)

---

## 1. What Is LWC?

**Lightning Web Components (LWC)** is Salesforce's modern UI framework, introduced in 2019 as the successor to the legacy Aura Components framework. LWC is built on **native web standards** — the same technologies that power modern browsers — rather than a proprietary abstraction layer.

### Core Characteristics

| Feature | Description |
|---------|-------------|
| **Standards-Based** | Built on native Web Components, ES6+ JavaScript, and HTML5 — no proprietary templating language |
| **Reactive Data Binding** | The UI re-renders automatically when tracked properties change, using `@track` and `@api` decorators |
| **Component-Based Architecture** | Each LWC is a self-contained, reusable UI unit with its own HTML, JavaScript, CSS, and metadata |
| **Salesforce Platform Integration** | Deep integration with Salesforce data via `@wire` adapters and Apex method calls |
| **Performance** | Runs natively in the browser with minimal overhead, significantly faster than the Aura framework |

### LWC vs. Aura Components

| Aspect | LWC | Aura Components |
|--------|-----|-----------------|
| Standard | Web Components (W3C) | Proprietary Aura framework |
| JavaScript | ES6+ Modules | Aura-specific syntax |
| Lifecycle | Native lifecycle callbacks | Custom Aura lifecycle hooks |
| Performance | High (native browser APIs) | Lower (abstraction overhead) |
| Recommended | ✅ Yes (current standard) | ❌ Legacy (use LWC instead) |

---

## 2. What Did You Build?

On Day 4, a **Placement Card LWC** was built — a custom UI component that displays a Candidate's placement summary information on the Salesforce record page.

### Component Features

- Displays candidate name, job title, placement status, and start date in a formatted card layout.
- Fetches live Placement data from Salesforce using a `@wire` service connected to an Apex method.
- Conditionally renders a status badge with color-coded styling (green for "Placed", amber for "Interviewing", grey for "Applied").
- Includes a "Refresh" button that re-fetches the latest record data without a full page reload.
- Displays a loading spinner while data is being fetched and an error message if the Apex call fails.

### Component Structure

```
force-app/main/default/lwc/placementCard/
├── placementCard.html          ← Component template (structure)
├── placementCard.js            ← Component controller (logic)
├── placementCard.css           ← Component styles (scoped styling)
└── placementCard.js-meta.xml   ← Metadata (targets, API version)
```

---

## 3. Which File Contains HTML?

**`placementCard.html`** contains the component's HTML template.

### Key Points About LWC HTML

- Every LWC HTML file must have a **single root `<template>` tag** — not `<html>` or `<div>`.
- LWC uses **template directives** for conditional rendering and list iteration:
  - `if:true={condition}` / `if:false={condition}` — Conditionally render elements.
  - `for:each={list} for:item="item"` — Iterate over a list of records.
- Data binding is done with **curly braces** `{propertyName}` — one-way binding from JS to HTML.
- **Event binding** uses `onclick={methodName}` syntax to attach JavaScript handlers.

```html
<template>
    <lightning-card title="Placement Summary" icon-name="standard:contact">
        <template if:true={placement}>
            <div class="slds-p-around_medium">
                <p><strong>Candidate:</strong> {placement.Candidate_Name__c}</p>
                <p><strong>Status:</strong> {placement.Status__c}</p>
                <p><strong>Start Date:</strong> {placement.Start_Date__c}</p>
            </div>
        </template>
        <template if:false={placement}>
            <p class="slds-p-around_medium">No placement data found.</p>
        </template>
    </lightning-card>
</template>
```

---

## 4. Which File Contains JavaScript?

**`placementCard.js`** contains the component's JavaScript controller.

### Key Points About LWC JavaScript

- The JS file exports a **class that extends `LightningElement`** — the base class for all LWC components.
- **Decorators** are used to define the component's reactive properties and connections:
  - `@api` — Exposes a property as a public attribute (configurable from parent components or App Builder).
  - `@track` — Marks a private property as reactive (deprecated for primitives in newer API versions; all properties are reactive by default now).
  - `@wire` — Connects a property or function to a Salesforce data source (Apex, UI API, etc.) reactively.
- **Lifecycle Hooks** (`connectedCallback`, `disconnectedCallback`, `renderedCallback`) control component behavior at different stages.

```javascript
import { LightningElement, api, wire } from 'lwc';
import getPlacementDetails from '@salesforce/apex/PlacementController.getPlacementDetails';

export default class PlacementCard extends LightningElement {
    @api recordId; // Passed automatically when placed on a record page

    placement;
    error;

    @wire(getPlacementDetails, { placementId: '$recordId' })
    wiredPlacement({ data, error }) {
        if (data) {
            this.placement = data;
            this.error = undefined;
        } else if (error) {
            this.error = error;
            this.placement = undefined;
        }
    }

    handleRefresh() {
        // Imperatively refresh the wire adapter
        refreshApex(this.wiredPlacement);
    }
}
```

---

## 5. What Did You Learn Today?

- **LWC is just modern JavaScript.** Unlike legacy Salesforce frameworks, LWC uses standard ES6 classes, modules, and decorators — skills that transfer directly to React, Vue, and other modern frameworks.
- **The `<template>` root tag is mandatory.** Every LWC HTML file wraps everything in `<template>`, not a standard HTML document structure.
- **`@wire` is the preferred data-fetching mechanism.** It creates a reactive binding between a component property and a data source, automatically re-fetching when input properties change.
- **`@api recordId` is the bridge to the record page.** When a component is deployed to a record page in App Builder, Salesforce automatically populates `recordId` with the current record's ID — no manual configuration needed.
- **CSS in LWC is scoped by default.** Styles defined in `placementCard.css` only apply to elements within `placementCard.html`, preventing style conflicts across components.
- **The component bundle is a unit of deployment.** All four files (HTML, JS, CSS, metadata XML) are deployed together as a single logical unit via the Salesforce CLI.

---

*Bootcamp Status: In Progress | Next: Day 5 — SOQL & SOSL Data Retrieval*
