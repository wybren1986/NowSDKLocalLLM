# ServiceNow Fluent SDK — Context Document for LLM

This document contains all the information you need to help build ServiceNow applications using the **Fluent SDK** (`@servicenow/sdk`). You work entirely from this document — you have no internet access and cannot fetch external documentation.

---

## What is the Fluent SDK?

The Fluent SDK lets you define ServiceNow artifacts (catalog items, flows, scripts, etc.) as **TypeScript files** that get deployed to a ServiceNow instance. Each artifact is defined by calling a constructor function (e.g. `CatalogItem(...)`, `Flow(...)`) and exporting the result.

Files use the `.now.ts` extension. Each exported value becomes a deployable artifact.

```typescript
// example.now.ts
import { CatalogItem } from '@servicenow/sdk/core'

export const myItem = CatalogItem({ ... })
```

---

## Project Structure

```
my-app/
  src/
    catalog-items/
      laptop-request.now.ts
    flows/
      laptop-fulfillment.now.ts
    modules/
      scripts/
        pre-insert.js
  now.config.json
```

`Now.ID['some_name']` generates a stable unique identifier for each artifact. Use `snake_case` for IDs.

---

## Part 1: Catalog Items

### When to use a Catalog Item vs Record Producer

| | Catalog Item | Record Producer |
|---|---|---|
| **Creates** | REQ + RITM + Tasks | A record in a target table (incident, change, etc.) |
| **Fulfillment** | Flow / Workflow | Server-side scripts |
| **Use when** | Ordering goods or services | Creating task records directly |
| **Examples** | "Request Laptop", "Software License" | "Report Incident", "Submit HR Case" |

**Rule of thumb:** Ordering something → Catalog Item. Creating a task record → Record Producer.

### Minimal Catalog Item

```typescript
import { CatalogItem } from '@servicenow/sdk/core'

export const myItem = CatalogItem({
  $id: Now.ID['my_item'],                        // required, unique ID
  name: 'My Item',                               // required
  catalogs: ['<sys_id of catalog>'],             // required
  categories: ['<sys_id of category>'],          // required
})
```

### Full Catalog Item with all common properties

```typescript
import { CatalogItem, SelectBoxVariable, MultiLineTextVariable } from '@servicenow/sdk/core'

export const laptopRequest = CatalogItem({
  $id: Now.ID['laptop_request'],
  name: 'Laptop Request',
  shortDescription: 'Request a new laptop',
  description: 'Submit a request for a new laptop.',

  catalogs: ['e0d08b13c3330100c8b837659bba8fb4'],
  categories: ['d258b953c611227a0146101fb1be7c31'],
  assignedTopics: ['782413a7c3053010069aec4b7d40ddf1'],  // for Employee Center visibility

  availableFor: ['<user_criteria sys_id>'],      // who can see this item
  notAvailableFor: ['<user_criteria sys_id>'],   // always overrides availableFor
  accessType: 'restricted',                      // 'restricted' | 'delegated'
  roles: ['itil'],

  active: true,
  availability: 'both',                          // 'desktopOnly' | 'mobileOnly' | 'both'
  requestMethod: 'order',                        // 'order' | 'request' | 'submit'

  pricingDetails: [{ amount: 1299, currencyType: 'USD', field: 'price' }],
  deliveryTime: { days: 7, hours: 0 },
  fulfillmentAutomationLevel: 'semiAutomated',   // 'unspecified'|'manual'|'semiAutomated'|'fullyAutomated'
  fulfillmentGroup: '<sys_user_group sys_id>',

  // Reference a flow by sys_id (existing platform flow):
  flow: '523da512c611228900811a37c97c2014',
  // OR reference a project-defined flow without circular import:
  // flow: Now.ref('sys_hub_flow', 'my_flow_id'),

  variables: {
    laptop_type: SelectBoxVariable({
      question: 'Laptop Type',
      mandatory: true,
      order: 100,
      choices: {
        standard: { label: 'Standard Laptop', sequence: 1 },
        developer: { label: 'Developer Workstation', sequence: 2 },
      },
    }),
    justification: MultiLineTextVariable({
      question: 'Business Justification',
      mandatory: true,
      order: 200,
    }),
  },

  // UI display options (all optional, all default false)
  hideAddToCart: false,
  hideAttachment: false,
  hideDeliveryTime: false,
  hideQuantitySelector: false,
  hideSaveAsDraft: false,
  hideSP: false,
})
```

### Catalog Item with recurring pricing

```typescript
export const softwareLicense = CatalogItem({
  $id: Now.ID['software_license'],
  name: 'Software License',
  catalogs: ['<sys_id>'],
  categories: ['<sys_id>'],
  pricingDetails: [
    { amount: 0,  currencyType: 'USD', field: 'price' },
    { amount: 99, currencyType: 'USD', field: 'recurring_price' },
  ],
  recurringFrequency: 'monthly',   // required when recurring_price is used
  variables: { /* ... */ },
})
```

### Circular dependency: Flow + CatalogItem

When a flow needs to use `getCatalogVariables` with this catalog item's variables, importing both would cause a circular dependency. Break the cycle with `Now.ref()`:

```typescript
// catalog-item.now.ts — uses Now.ref(), does NOT import the flow
export const myItem = CatalogItem({
  $id: Now.ID['my_item'],
  flow: Now.ref('sys_hub_flow', 'my_flow'),   // NO import needed
  variables: { urgency: SelectBoxVariable({ ... }) },
})

// flow.now.ts — imports catalog item for getCatalogVariables
import { myItem } from '../catalog-items/my-item.now'
export const myFlow = Flow(...)
```

---

## Part 2: Catalog Variables

### Common properties for all variable types

| Property | Type | Description |
|---|---|---|
| `question` | string | **Required.** Label shown to user |
| `order` | number | Display order — always set, use increments of 100 |
| `mandatory` | boolean | Required field. Default: `false` |
| `readOnly` | boolean | Not editable. Default: `false` |
| `hidden` | boolean | Not visible. Default: `false` |
| `tooltip` | string | Hover help text |
| `exampleText` | string | Placeholder text |
| `instructions` | string | Inline help text |
| `defaultValue` | string | Pre-filled value |
| `width` | 25\|50\|75\|100 | Field width percentage |
| `mapToField` | boolean | Map to table field (Record Producers only) |
| `field` | string | Target field name when `mapToField: true` |

### Text Variables

```typescript
import {
  SingleLineTextVariable,
  MultiLineTextVariable,
  WideSingleLineTextVariable,
  EmailVariable,
  UrlVariable,
  IpAddressVariable,
  MaskedVariable,
} from '@servicenow/sdk/core'

variables: {
  name:      SingleLineTextVariable({ question: 'Full Name', order: 100 }),
  notes:     MultiLineTextVariable({ question: 'Notes', order: 200 }),
  wide:      WideSingleLineTextVariable({ question: 'Summary', order: 300 }),
  email:     EmailVariable({ question: 'Email', order: 400 }),
  website:   UrlVariable({ question: 'URL', order: 500 }),
  ip:        IpAddressVariable({ question: 'IP Address', order: 600 }),
  password:  MaskedVariable({ question: 'Password', order: 700, useEncryption: true }),
}
```

### Choice Variables

```typescript
import { SelectBoxVariable, MultipleChoiceVariable, YesNoVariable } from '@servicenow/sdk/core'

variables: {
  priority: SelectBoxVariable({
    question: 'Priority',
    order: 100,
    mandatory: true,
    choices: {
      high:   { label: 'High',   sequence: 1 },
      medium: { label: 'Medium', sequence: 2 },
      low:    { label: 'Low',    sequence: 3 },
    },
  }),
  direction: MultipleChoiceVariable({
    question: 'Preferred Contact',
    order: 200,
    choiceDirection: 'down',  // 'down' | 'across'
    choices: {
      email: { label: 'Email', sequence: 1 },
      phone: { label: 'Phone', sequence: 2 },
    },
  }),
  urgent: YesNoVariable({
    question: 'Is this urgent?',
    order: 300,
    includeNone: true,
  }),
}
```

### Date/Time Variables

```typescript
import { DateVariable, DateTimeVariable } from '@servicenow/sdk/core'

variables: {
  start_date: DateVariable({ question: 'Start Date', order: 100 }),
  scheduled:  DateTimeVariable({ question: 'Scheduled Date/Time', order: 200 }),
}
```

### Reference Variables

```typescript
import { ReferenceVariable, RequestedForVariable } from '@servicenow/sdk/core'

variables: {
  user: ReferenceVariable({
    question: 'User',
    order: 100,
    referenceTable: 'sys_user',
    referenceQualCondition: 'active=true',     // simple filter
    // OR:
    useReferenceQualifier: 'dynamic',
    dynamicRefQual: '<dynamic_filter_sys_id>',
    // OR:
    useReferenceQualifier: 'advanced',
    referenceQual: 'active=true^department=IT',
  }),
  requested_for: RequestedForVariable({
    question: 'Requested For',
    order: 200,
    enableAlsoRequestFor: true,
    rolesToUseAlsoRequestFor: ['admin', 'itil'],
  }),
}
```

### Checkbox Variable

```typescript
import { CheckboxVariable } from '@servicenow/sdk/core'

variables: {
  premium: CheckboxVariable({
    question: 'Add Premium Support',
    order: 100,
    selectionRequired: true,
    pricingDetails: [{ amount: 100, currencyType: 'USD', field: 'price_if_checked' }],
  }),
}
```

### Variable Sets (reusable groups)

```typescript
import { VariableSet, SingleLineTextVariable } from '@servicenow/sdk/core'

export const contactInfoSet = VariableSet({
  $id: Now.ID['contact_info_set'],
  name: 'Contact Information',
  type: 'singleRow',  // 'singleRow' | 'multiRow'
  variables: {
    phone: SingleLineTextVariable({ question: 'Phone', order: 100 }),
    email: EmailVariable({ question: 'Email', order: 200 }),
  },
})

// In a CatalogItem:
export const myItem = CatalogItem({
  ...
  variableSets: [
    { variableSet: contactInfoSet, order: 100 },
  ],
})
```

### UI Policies (show/hide/mandatory logic)

Use for simple show/hide, mandatory, or read-only rules. Prefer over Client Scripts when possible.

```typescript
import { CatalogUiPolicy } from '@servicenow/sdk/core'

export const showJustification = CatalogUiPolicy({
  $id: Now.ID['show_justification_policy'],
  catalogItem: myItem,
  conditions: `priority=high`,
  actions: {
    justification: { visible: true, mandatory: true },
    other_field:   { visible: false },
  },
  runScripts: true,
  reverseIfFalse: true,
})
```

### Client Scripts (complex validation / dynamic logic)

Use when you need: validation, async calls (GlideAjax), calculations, or form submission control.

```typescript
import { CatalogClientScript } from '@servicenow/sdk/core'

export const validateDate = CatalogClientScript({
  $id: Now.ID['validate_date_script'],
  catalogItem: myItem,
  type: 'onChange',     // 'onLoad' | 'onChange' | 'onSubmit'
  fieldName: 'start_date',
  script: validateDateScript,
})
```

```javascript
// validate-date.js
export function validateDateScript(control, oldValue, newValue, isLoading) {
  if (isLoading) return;  // Always guard onChange scripts

  var today = new GlideDateTime();
  var selected = new GlideDateTime(newValue);
  if (selected < today) {
    g_form.showErrorBox('start_date', 'Date cannot be in the past');
    g_form.setValue('start_date', '');
  }
}
```

**Rules:**
- Always start `onChange` with `if (isLoading) return;`
- Never use `GlideAjax` in `onSubmit` (async issues)
- Return `false` from `onSubmit` to block submission
- Never manipulate DOM directly — always use `g_form` API
- Use object references in `.now.ts` (`catalogItem.variables.field_name`), strings in JS scripts (`g_form.getValue('field_name')`)

---

## Part 3: Record Producers

```typescript
import { CatalogItemRecordProducer } from '@servicenow/sdk/core'

export const incidentProducer = CatalogItemRecordProducer({
  $id: Now.ID['report_incident'],
  name: 'Report an Incident',
  table: 'incident',                             // target table, required
  catalogs: ['<sys_id>'],
  categories: ['<sys_id>'],

  variables: {
    short_description: SingleLineTextVariable({
      question: 'Brief Summary',
      mandatory: true,
      mapToField: true,
      field: 'short_description',
      order: 100,
    }),
    urgency: SelectBoxVariable({
      question: 'Urgency',
      mandatory: true,
      mapToField: true,
      field: 'urgency',
      choices: {
        '1': { label: 'High',   sequence: 1 },
        '2': { label: 'Medium', sequence: 2 },
        '3': { label: 'Low',    sequence: 3 },
      },
      order: 200,
    }),
  },

  script: preInsertScript,          // runs BEFORE record creation
  postInsertScript: postInsertScript, // runs AFTER record creation
  redirectUrl: 'generatedRecord',   // 'generatedRecord' | 'catalogHomePage'
  allowEdit: true,
})
```

```javascript
// pre-insert.js — never call current.update() here
import { gs } from '@servicenow/glide'
export function preInsertScript(current, producer) {
  current.caller_id = gs.getUserID();
  current.contact_type = 'self-service';
}

// post-insert.js — safe to call current.update() here
export function postInsertScript(current, producer) {
  current.work_notes = 'Created via catalog at ' + gs.nowDateTime();
  current.update();
}
```

**Record Producer rules:**
- Only use for task tables: `incident`, `change_request`, `problem` — NOT for `sc_request`, `sc_req_item`, `sc_task`
- Never call `current.update()` or `current.insert()` in the `script` (pre-insert)
- Never call `current.setAbortAction()`

---

## Part 4: Flows

Flows define automated fulfillment logic. They consist of: configuration, a trigger (when), and actions (what).

### Basic flow structure

```typescript
import { Flow, wfa, trigger, action } from '@servicenow/sdk/automation'

export const myFlow = Flow(
  {
    $id: Now.ID['my_flow'],
    name: 'My Flow',
    description: 'What this flow does',
    runAs: 'system',     // 'system' | 'user'
    flowPriority: 'HIGH', // 'LOW' | 'MEDIUM' | 'HIGH'
  },
  wfa.trigger(trigger.record.created, { $id: Now.ID['trigger'] }, {
    table: 'incident',
    condition: 'priority=1',
    run_flow_in: 'background',
  }),
  (_params) => {
    // actions go here
  }
)
```

> **Important:** After deploying, activate the flow in Flow Designer — flows deploy as draft.

> **Important:** Do NOT import `Time`, `Duration`, or `TemplateValue` — they are available globally.

### Flow Configuration Properties

| Property | Type | Default | Description |
|---|---|---|---|
| `$id` | string | — | Required. `Now.ID['name']` |
| `name` | string | — | Required. Display name |
| `description` | string | — | Flow purpose |
| `runAs` | `'system'\|'user'` | `'user'` | Execution context. Use `'system'` when automation must complete regardless of user permissions |
| `runWithRoles` | string[] | `[]` | Required roles |
| `flowPriority` | `'LOW'\|'MEDIUM'\|'HIGH'` | `'MEDIUM'` | Use `'HIGH'` for time-sensitive user-facing flows, `'LOW'` for batch/cleanup |

### Data Pills

`wfa.dataPill(expression, type)` wraps values for Flow Designer serialization.

```typescript
// Trigger fields
wfa.dataPill(_params.trigger.current.short_description, 'string')
wfa.dataPill(_params.trigger.current.priority, 'integer')
wfa.dataPill(_params.trigger.current, 'reference')

// Dot-walking through references
wfa.dataPill(_params.trigger.current.assigned_to.email, 'string')
wfa.dataPill(_params.trigger.current.assigned_to.manager.name, 'string')

// Action outputs
wfa.dataPill(createResult.record, 'reference')
wfa.dataPill(lookupResult.Records, 'array.object')

// In conditions (use template literal)
condition: `${wfa.dataPill(_params.trigger.current.priority, 'string')}=1`

// In log messages (template literal supported)
log_message: `Incident ${wfa.dataPill(_params.trigger.current.number, 'string')} created`
```

Common types: `'string'`, `'integer'`, `'boolean'`, `'reference'`, `'datetime'`, `'glide_date_time'`, `'array.object'`, `'records'`, `'html'`

**Critical rule:** Data pills must be used directly in action parameters — never assign to variables first.

```typescript
// WRONG
const id = wfa.dataPill(result.record, 'reference')
wfa.action(action.core.updateRecord, ..., { record: id })

// CORRECT
wfa.action(action.core.updateRecord, ..., {
  record: wfa.dataPill(result.record, 'reference')
})
```

### Triggers

#### Record triggers

```typescript
// When a record is created
wfa.trigger(trigger.record.created, { $id: Now.ID['t'] }, {
  table: 'incident',
  condition: 'priority=1',
  run_flow_in: 'background',  // 'any' | 'background' | 'foreground'
})

// When a record is updated
wfa.trigger(trigger.record.updated, { $id: Now.ID['t'] }, {
  table: 'incident',
  condition: 'state=6',
  trigger_strategy: 'once',   // 'once' | 'unique_changes' | 'every' | 'always'
})

// When a record is created or updated
wfa.trigger(trigger.record.createdOrUpdated, { $id: Now.ID['t'] }, {
  table: 'incident',
})
```

Access: `_params.trigger.current`, `_params.trigger.current.field_name`

#### Scheduled triggers

```typescript
// Daily at 02:00 UTC
wfa.trigger(trigger.scheduled.daily, { $id: Now.ID['t'] }, {
  time: Time({ hours: 2, minutes: 0, seconds: 0 }, 'UTC')
})

// Weekly on Monday at 09:00 UTC
wfa.trigger(trigger.scheduled.weekly, { $id: Now.ID['t'] }, {
  day_of_week: 1,  // 1=Monday ... 7=Sunday
  time: Time({ hours: 9, minutes: 0, seconds: 0 }, 'UTC')
})

// Monthly on the 1st
wfa.trigger(trigger.scheduled.monthly, { $id: Now.ID['t'] }, {
  day_of_month: 1,
  time: Time({ hours: 0, minutes: 0, seconds: 0 }, 'UTC')
})

// Every 15 minutes
wfa.trigger(trigger.scheduled.repeat, { $id: Now.ID['t'] }, {
  repeat: Duration({ minutes: 15 })
})

// Once at a specific time
wfa.trigger(trigger.scheduled.runOnce, { $id: Now.ID['t'] }, {
  run_in: '2026-06-01 09:00:00'
})
```

#### Service Catalog trigger

```typescript
wfa.trigger(trigger.application.serviceCatalog, { $id: Now.ID['t'] }, {
  run_flow_in: 'background'
})
// Access: _params.trigger.request_item, _params.trigger.request_item.short_description
```

### Actions — Record operations

```typescript
// Create a record
const newRecord = wfa.action(action.core.createRecord,
  { $id: Now.ID['create_inc'] },
  {
    table_name: 'incident',
    values: TemplateValue({
      short_description: wfa.dataPill(_params.trigger.current.short_description, 'string'),
      priority: '1',
      caller_id: wfa.dataPill(_params.trigger.current.caller_id, 'reference'),
    }),
  }
)
// Output: newRecord.record (reference)

// Update a record
wfa.action(action.core.updateRecord,
  { $id: Now.ID['update_inc'] },
  {
    table_name: 'incident',
    record: wfa.dataPill(_params.trigger.current, 'reference'),
    values: TemplateValue({
      state: 2,
      assignment_group: wfa.dataPill(groupResult.Record, 'reference'),
    }),
  }
)

// Look up a single record — note uppercase output 'Record'
const found = wfa.action(action.core.lookUpRecord,
  { $id: Now.ID['find_group'] },
  { table: 'sys_user_group', conditions: 'name=Service Desk' }
)
// Output: found.Record (reference), found.status

// Look up multiple records — note uppercase 'Records' and 'Count'
const results = wfa.action(action.core.lookUpRecords,
  { $id: Now.ID['find_open'] },
  { table: 'incident', conditions: 'active=true^priority=1', max_results: 50 }
)
// Output: results.Records (array), results.Count (integer)

// Bulk update
wfa.action(action.core.updateMultipleRecords,
  { $id: Now.ID['bulk_update'] },
  {
    table_name: 'incident',
    conditions: 'state=1^active=true',
    field_values: TemplateValue({ priority: '2' }),  // note: field_values not values
  }
)

// Delete a record
wfa.action(action.core.deleteRecord,
  { $id: Now.ID['delete'] },
  { record: wfa.dataPill(found.Record, 'reference') }
)
```

### Actions — Communication

```typescript
// Send email (ah_body only accepts static strings, NOT data pills)
wfa.action(action.core.sendEmail,
  { $id: Now.ID['send_email'] },
  {
    ah_to: 'user@example.com',
    ah_subject: `Incident ${wfa.dataPill(_params.trigger.current.number, 'string')} assigned`,
    ah_body: 'Please check your assignment queue.',   // static only!
    record: wfa.dataPill(_params.trigger.current, 'reference'),
    table_name: 'incident',
  }
)

// Log
wfa.action(action.core.log,
  { $id: Now.ID['log'] },
  {
    log_level: 'info',
    log_message: `Processing ${wfa.dataPill(_params.trigger.current.number, 'string')}`,
  }
)
```

### Actions — Approvals

```typescript
const approval = wfa.action(action.core.askForApproval,
  { $id: Now.ID['approval'] },
  {
    record: wfa.dataPill(_params.trigger.current, 'reference'),
    table: 'change_request',
    approval_conditions: wfa.approvalRules({
      conditionType: 'OR',
      ruleSets: [{
        action: 'Approves',
        conditionType: 'AND',
        rules: [[{
          ruleType: 'Any',  // 'Any' | 'All' | 'Count' | 'Percent'
          users: ['<user_sys_id>'],
          groups: [],
          manual: false,
        }]],
      }],
    }),
  }
)
// Output: approval.approval_state ('approved' | 'rejected' | 'requested' | 'not_required' | 'cancelled')
```

### Flow Logic — If/Else

```typescript
wfa.flowLogic.if(
  { $id: Now.ID['check_priority'] },
  `${wfa.dataPill(_params.trigger.current.priority, 'string')}=1`,
  () => {
    // actions for high priority
    wfa.action(action.core.log, { $id: Now.ID['log_high'] }, { log_level: 'info', log_message: 'High priority' })
  },
  [
    {
      condition: `${wfa.dataPill(_params.trigger.current.priority, 'string')}=2`,
      body: () => {
        wfa.action(action.core.log, { $id: Now.ID['log_med'] }, { log_level: 'info', log_message: 'Medium' })
      },
    },
  ],
  () => {
    // else
    wfa.action(action.core.log, { $id: Now.ID['log_low'] }, { log_level: 'info', log_message: 'Low' })
  }
)
```

**Important:** Flow logic conditions only support data pill comparisons with static values. JavaScript functions like `gs.daysAgoStart()` are NOT supported in conditions.

### Flow Logic — forEach

```typescript
const records = wfa.action(action.core.lookUpRecords, { $id: Now.ID['find'] }, {
  table: 'incident', conditions: 'active=true'
})

wfa.flowLogic.forEach(
  wfa.dataPill(records.Records, 'array.object'),
  { $id: Now.ID['each'] },
  (item) => {
    wfa.action(action.core.updateRecord, { $id: Now.ID['update'] }, {
      table_name: 'incident',
      record: wfa.dataPill(item, 'reference'),
      values: TemplateValue({ state: 7 }),
    })
  }
)
```

### Wait actions

```typescript
// Wait until a condition is met
wfa.action(action.core.waitForCondition,
  { $id: Now.ID['wait'] },
  {
    record: `${wfa.dataPill(_params.trigger.current, 'reference')}`,
    conditions: 'state=3',
    table_name: 'change_request',
    timeout_flag: true,
    timeout_duration: Duration({ hours: 48 }),
  }
)
```

### Service Catalog flow — reading variables

```typescript
import { myCatalogItem } from '../catalog-items/my-item.now'

export const fulfillmentFlow = Flow(
  { $id: Now.ID['fulfill_flow'], name: 'Fulfill My Item', runAs: 'system' },
  wfa.trigger(trigger.application.serviceCatalog, { $id: Now.ID['t'] }, {}),
  (_params) => {
    const vars = wfa.action(action.core.getCatalogVariables,
      { $id: Now.ID['get_vars'] },
      {
        template_catalog_item: `${myCatalogItem}`,
        catalog_variables: [
          myCatalogItem.variables.laptop_type,
          myCatalogItem.variables.justification,
        ],
      }
    )
    // use vars.laptop_type, vars.justification as data pills
  }
)
```

---

## Critical Rules Summary

1. **Variables:** Always set `order`, always use `snake_case` names, never use a variable name that matches a target table field name.
2. **Catalogs/Categories:** Required on all catalog items and record producers.
3. **Record Producers:** Never `current.update()` in pre-insert script. Only for task tables (incident, change_request, problem).
4. **Flows are declarative:** No JavaScript abstractions, no assigning data pills to variables. Use data pills directly.
5. **Template literals in flows:** Only supported in `ah_subject` and `log_message`. Static strings required in `ah_body` and `TemplateValue` fields.
6. **lookUpRecord/lookUpRecords outputs are uppercase:** `Record`, `Records`, `Count`.
7. **createRecord/updateRecord use `values:`**, while **createTask/updateMultipleRecords use `field_values:`**.
8. **Don't import:** `Time`, `Duration`, `TemplateValue` — they are global.
9. **Circular dependency:** CatalogItem → `Now.ref()` to flow; Flow → `import` from catalog item.
10. **Activate flows after deploy:** Flows deploy as draft; activate in Flow Designer.
