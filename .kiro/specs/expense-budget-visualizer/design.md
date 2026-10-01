# Design Document: Expense & Budget Visualizer

## Overview

The Expense & Budget Visualizer is a self-contained, client-side web application delivered as a single HTML file. It allows users to record personal expense transactions, view a running total balance, and see a live pie chart of spending distribution by category. All data is persisted in the browser's Local Storage API, enabling the app to survive page reloads without any backend infrastructure.

### Key Design Goals

- **Zero dependencies on a server**: the app ships as one `.html` file with inline CSS, inline JS modules, and a single CDN-loaded chart library.
- **Immediate feedback**: every add/delete operation updates the UI synchronously within the same event-loop tick (well under the 100 ms budget).
- **Separation of concerns**: state management, DOM rendering, storage I/O, and chart rendering are isolated into distinct logical modules inside the same file to keep the code maintainable without a build system.
- **Graceful degradation**: Local Storage failures are caught and surfaced as non-blocking toast messages; the app remains fully usable in memory.

---

## Architecture

The application follows a unidirectional data-flow pattern adapted for vanilla JavaScript:

```
User Interaction (DOM Events)
        │
        ▼
   Controller Layer
  (event handlers in app.js)
        │
        ├──► State Store (in-memory array of transactions)
        │           │
        │           ▼
        │      Storage Module  ──► localStorage
        │
        ▼
   Render Pipeline
  ┌─────────────────────────────────────┐
  │  renderTransactionList()            │
  │  renderTotalBalance()               │
  │  renderChart()                      │
  └─────────────────────────────────────┘
```

**No two-way data binding**: state flows in one direction. An event mutates the in-memory state array, then a full re-render is triggered. This keeps the render functions as pure projections of state onto DOM, which simplifies debugging and testing.

### Module Breakdown (inside a single `<script type="module">` block)

| Module | Responsibility |
|---|---|
| `storage.js` (inline) | `load()`, `save(transactions)` — wraps localStorage with error handling |
| `validator.js` (inline) | `validate(formData)` — returns `{ valid, errors }` |
| `state.js` (inline) | In-memory `transactions[]` array; `addTransaction()`, `deleteTransaction()` |
| `render.js` (inline) | DOM update functions: list, balance, chart |
| `chart.js` (inline) | Thin wrapper around Chart.js instance; `updateChart(transactions)` |
| `app.js` (inline) | Bootstraps the app, wires up event listeners, orchestrates the pipeline |

Since the entire application lives in one HTML file, these "modules" are implemented as separate `const` objects or immediately-invoked function groups within a single `<script type="module">` tag, ensuring a clean namespace without polluting `window`.

---

## Components and Interfaces

### HTML Structure

```
<body>
  <header>
    <h1>Expense & Budget Visualizer</h1>
    <section id="balance-section">   <!-- Total Balance -->
  </header>

  <main>
    <section id="form-section">      <!-- Input Form -->
    <section id="chart-section">     <!-- Pie Chart -->
    <section id="list-section">      <!-- Transaction List -->
  </main>

  <div id="toast-container" aria-live="polite">  <!-- Non-blocking notifications -->
</body>
```

### Input Form Component

**Element**: `<form id="transaction-form">`

| Field | Element | Validation Rules |
|---|---|---|
| Item Name | `<input type="text" id="item-name">` | 1–100 non-whitespace-only characters |
| Amount | `<input type="number" id="amount" step="0.01">` | 0.01 ≤ value ≤ 999,999,999.99 |
| Category | `<select id="category">` | Must select Food, Transport, or Fun |
| Submit | `<button type="submit">Add</button>` | — |

Each field is paired with an `<span class="error-msg" aria-live="assertive">` element for inline validation errors.

**Submit handler interface**:
```javascript
form.addEventListener('submit', (e) => {
  e.preventDefault();
  const result = Validator.validate({ name, amount, category });
  if (!result.valid) { renderErrors(result.errors); return; }
  const tx = State.addTransaction({ name, amount, category });
  Storage.save(State.getAll());
  renderAll();
  form.reset();
});
```

### Transaction List Component

**Element**: `<ul id="transaction-list">` — each item rendered as `<li data-id="{id}">`.

Each list item contains:
- Item name (truncated to 100 chars with CSS `text-overflow: ellipsis`)
- Amount formatted as `$0.00` using `toLocaleString('en-US', { style: 'currency', currency: 'USD' })`
- Category badge
- Delete button: `<button class="delete-btn" aria-label="Delete {name}">✕</button>`

Empty state: when `transactions.length === 0`, the list shows a single `<li class="empty-state">No transactions yet.</li>`.

### Total Balance Component

**Element**: `<div id="total-balance">` — updated via `textContent`.

```javascript
function renderTotalBalance(transactions) {
  const total = transactions.reduce((sum, tx) => sum + tx.amount, 0);
  balanceEl.textContent = total.toLocaleString('en-US', { style: 'currency', currency: 'USD' });
}
```

### Pie Chart Component

**Library**: [Chart.js v4](https://www.chartjs.org/) loaded via CDN:
```html
<script src="https://cdn.jsdelivr.net/npm/chart.js@4.4.1/dist/chart.umd.min.js"></script>
```

**Element**: `<canvas id="spending-chart"></canvas>` inside `#chart-section`.

The chart instance is created once on initialization and updated (not recreated) on every state change using `chart.data = newData; chart.update()`. This avoids flickering and memory leaks from repeatedly creating Chart instances.

```javascript
// Chart initialization
const chart = new Chart(ctx, {
  type: 'pie',
  data: buildChartData(transactions),
  options: {
    responsive: true,
    plugins: {
      legend: { position: 'bottom' },
      tooltip: { enabled: true },
    },
  },
});

// Update function
function updateChart(transactions) {
  const aggregated = aggregateByCategory(transactions);
  chart.data = buildChartData(aggregated);
  chart.update();
}
```

When no transactions exist, the `<canvas>` is hidden (`display: none`) and a `<p id="chart-empty-msg">` placeholder is shown instead.

**Category color palette** (fixed, deterministic):

| Category | Color |
|---|---|
| Food | `#FF6384` |
| Transport | `#36A2EB` |
| Fun | `#FFCE56` |

### Toast Notification Component

**Element**: `<div id="toast-container" aria-live="polite" aria-atomic="true">`

Non-blocking error messages (storage failures) are shown as toast elements that auto-dismiss after 4 seconds via `setTimeout`. They do not block user interaction.

```javascript
function showToast(message, type = 'error') {
  const toast = document.createElement('div');
  toast.className = `toast toast--${type}`;
  toast.textContent = message;
  toastContainer.appendChild(toast);
  setTimeout(() => toast.remove(), 4000);
}
```

---

## Data Models

### Transaction Object

```javascript
/**
 * @typedef {Object} Transaction
 * @property {string}  id        - UUID generated at creation time (crypto.randomUUID())
 * @property {string}  name      - Item name (1–100 characters, trimmed)
 * @property {number}  amount    - Positive numeric value (0.01–999,999,999.99)
 * @property {string}  category  - One of: 'Food' | 'Transport' | 'Fun'
 * @property {number}  createdAt - Unix timestamp (Date.now()) for ordering
 */
```

### In-Memory State

```javascript
let transactions = []; // Transaction[]
// Sorted descending by createdAt at render time (not stored sorted)
```

### Local Storage Schema

Key: `"expense-budget-visualizer-transactions"`

Value: JSON-serialized array of `Transaction` objects.

```json
[
  {
    "id": "550e8400-e29b-41d4-a716-446655440000",
    "name": "Lunch",
    "amount": 12.50,
    "category": "Food",
    "createdAt": 1700000000000
  }
]
```

**Storage operations**:

```javascript
const Storage = {
  KEY: 'expense-budget-visualizer-transactions',

  load() {
    try {
      const raw = localStorage.getItem(this.KEY);
      return raw ? JSON.parse(raw) : [];
    } catch (e) {
      return null; // signals parse or unavailability error
    }
  },

  save(transactions) {
    try {
      localStorage.setItem(this.KEY, JSON.stringify(transactions));
      return true;
    } catch (e) {
      return false; // signals write failure (e.g. quota exceeded)
    }
  },
};
```

### Validation Error Model

```javascript
/**
 * @typedef {Object} ValidationResult
 * @property {boolean} valid
 * @property {{ name?: string, amount?: string, category?: string }} errors
 */
```

### Chart Data Model

```javascript
// Intermediate aggregation shape used to build Chart.js data
/**
 * @typedef {Object} CategoryAggregate
 * @property {string[]} labels   - Category names present in transactions
 * @property {number[]} data     - Total amount per category (same order as labels)
 * @property {string[]} colors   - Background colors (same order as labels)
 */
```

---

## Correctness Properties

*A property is a characteristic or behavior that should hold true across all valid executions of a system — essentially, a formal statement about what the system should do. Properties serve as the bridge between human-readable specifications and machine-verifiable correctness guarantees.*

### Property 1: Invalid item names are rejected

*For any* string that is either empty or composed entirely of whitespace characters, submitting it as the item name SHALL be rejected by the Validator, the `errors.name` field SHALL be non-empty, and the transaction list SHALL remain unchanged.

**Validates: Requirements 1.2, 1.5**

---

### Property 2: Valid form submission appends one transaction at the top

*For any* valid combination of item name, amount in [0.01, 999,999,999.99], and a valid category, calling `State.addTransaction()` SHALL increase the transaction list length by exactly 1, and when the list is sorted by `createdAt` descending the newly added transaction SHALL appear at index 0.

**Validates: Requirements 1.4, 2.3**

---

### Property 3: Balance invariant — balance equals sum of all amounts

*For any* array of transactions (empty, single, or multiple), `computeBalance(transactions)` SHALL return the exact arithmetic sum of all `amount` values. This invariant holds unconditionally — after initialization, after any add, and after any delete.

**Validates: Requirements 4.1, 4.2, 4.3, 4.4**

---

### Property 4: Storage round-trip preserves transaction data

*For any* transaction object, serializing the transactions array to JSON via `Storage.save()` and then deserializing it via `Storage.load()` SHALL produce a transaction object with identical `id`, `name`, `amount`, `category`, and `createdAt` fields as the original.

**Validates: Requirements 6.2, 6.3**

---

### Property 5: Chart data reflects current transaction categories

*For any* non-empty set of transactions, the result of `aggregateByCategory(transactions)` SHALL contain exactly one entry per distinct category that has at least one transaction, and each entry's percentage SHALL equal `(categoryTotal / grandTotal * 100)` rounded to one decimal place.

**Validates: Requirements 5.1, 5.2, 5.3, 5.4**

---

### Property 6: Rendered transaction item contains all required fields and a delete control

*For any* transaction object, the HTML produced by the transaction list render function SHALL contain the item name (or its first 100 characters), the amount formatted to 2 decimal places with a currency symbol, the category label, and a delete control element.

**Validates: Requirements 2.1, 3.1**

---

### Property 7: Deletion removes transaction from state and storage

*For any* transaction present in the in-memory state, calling `State.deleteTransaction(id)` followed by `Storage.save()` SHALL result in neither the in-memory state nor the deserialized Local Storage containing a transaction with that `id`, and `computeBalance()` SHALL reflect the removal.

**Validates: Requirements 3.2, 3.3**

---

### Property 8: Out-of-range amounts are rejected

*For any* numeric string representing a value outside the closed interval [0.01, 999,999,999.99] (including zero, negative values, and values above the maximum), `Validator.validate()` SHALL return `valid: false` with a non-empty `errors.amount` field.

**Validates: Requirements 1.1, 1.2, 1.3**

---

## Error Handling

### Local Storage Unavailable on Load

**Trigger**: `localStorage.getItem()` throws (private browsing mode with storage disabled, or quota exceeded).

**Behavior**: `Storage.load()` returns `null`. `app.js` checks for `null` and calls `showToast('Saved data could not be loaded. Starting fresh.')`. App initializes with an empty `transactions` array.

### Local Storage Write Failure

**Trigger**: `localStorage.setItem()` throws (storage quota exceeded).

**Behavior**: `Storage.save()` returns `false`. `app.js` checks the return value and calls `showToast('Your change could not be saved persistently.')`. The in-memory state is still updated, so the UI remains consistent for the current session.

**Rationale**: The transaction is NOT rolled back from the in-memory state on save failure. Rolling it back would be confusing for the user (they just saw it added). The toast informs them of the persistence issue without disrupting their workflow.

### Validation Errors

**Trigger**: User submits the form with missing or invalid fields.

**Behavior**: Errors are injected into the `<span class="error-msg">` elements adjacent to the offending fields. Valid fields retain their values. Focus is moved to the first invalid field for accessibility.

### Chart.js Load Failure

**Trigger**: CDN is unreachable (offline scenario).

**Behavior**: The `<canvas>` element stays hidden and a fallback message reads: "Chart unavailable — check your internet connection." The rest of the app (form, list, balance) continues to function normally without Chart.js.

---

## Testing Strategy

Since this is a standalone Vanilla JavaScript application with no build tooling, tests are structured as follows:

### Unit Tests (Example-Based)

Targeting the pure logic modules with no DOM dependencies:

| Module | Test Scenarios |
|---|---|
| `Validator.validate()` | Blank name, whitespace-only name, amount = 0, amount = 0.005 (rounds to 0.01), amount = 1e9, valid form |
| `Storage.load()` | Valid JSON, malformed JSON, missing key, quota exception |
| `Storage.save()` | Normal save, quota exceeded exception |
| `aggregateByCategory()` | Empty list, single category, all three categories, deleted last item in a category |
| Balance computation | Empty list, single transaction, multiple transactions, float precision |

Unit tests can be written using plain `console.assert()` calls in a separate `tests.html` file or with a lightweight runner like [uvu](https://github.com/lukeed/uvu) if a minimal Node.js test runner is acceptable.

### Property-Based Tests

The feature involves pure functions with well-defined input/output behavior, making it a good candidate for property-based testing. The recommended library is [fast-check](https://fast-check.dev/) for JavaScript, loadable via CDN in a `tests.html` harness.

**Minimum 100 iterations per property test.** Each test is tagged with a comment referencing the design property it validates.

**Property Test Implementations**:

```javascript
// Feature: expense-budget-visualizer, Property 1: Invalid item names are rejected
fc.assert(fc.property(
  fc.oneof(fc.constant(''), fc.stringMatching(/^\s+$/)),
  (name) => {
    const result = Validator.validate({ name, amount: '10', category: 'Food' });
    return !result.valid && result.errors.name !== undefined;
  }
), { numRuns: 100 });

// Feature: expense-budget-visualizer, Property 2: Valid submission appends one transaction at top
fc.assert(fc.property(validTransactionInputArb, fc.array(validTransactionArb), (input, existing) => {
  const before = [...existing];
  const after = addTransaction(existing, input);
  if (after.length !== before.length + 1) return false;
  const sorted = [...after].sort((a, b) => b.createdAt - a.createdAt);
  return sorted[0].name === input.name;
}), { numRuns: 100 });

// Feature: expense-budget-visualizer, Property 3: Balance invariant — balance equals sum of all amounts
fc.assert(fc.property(fc.array(validTransactionArb), (txs) => {
  const expected = txs.reduce((sum, tx) => sum + tx.amount, 0);
  return computeBalance(txs) === expected;
}), { numRuns: 100 });

// Feature: expense-budget-visualizer, Property 4: Storage round-trip preserves transaction data
fc.assert(fc.property(validTransactionArb, (tx) => {
  const stored = JSON.parse(JSON.stringify([tx]));
  return stored[0].id === tx.id
    && stored[0].name === tx.name
    && stored[0].amount === tx.amount
    && stored[0].category === tx.category
    && stored[0].createdAt === tx.createdAt;
}), { numRuns: 100 });

// Feature: expense-budget-visualizer, Property 5: Chart data reflects current transaction categories
fc.assert(fc.property(fc.array(validTransactionArb, { minLength: 1 }), (txs) => {
  const agg = aggregateByCategory(txs);
  const distinctCategories = new Set(txs.map(t => t.category));
  if (agg.labels.length !== distinctCategories.size) return false;
  const total = txs.reduce((s, t) => s + t.amount, 0);
  return agg.data.every((val, i) => {
    const expected = Math.round((val / total) * 1000) / 10;
    return agg.percentages[i] === expected;
  });
}), { numRuns: 100 });

// Feature: expense-budget-visualizer, Property 6: Rendered item contains all required fields + delete control
fc.assert(fc.property(validTransactionArb, (tx) => {
  const html = renderTransactionItem(tx);
  return html.includes(tx.name.slice(0, 100))
    && html.includes(tx.category)
    && /[\$£€][\d,]+\.\d{2}/.test(html)
    && html.includes('delete-btn');
}), { numRuns: 100 });

// Feature: expense-budget-visualizer, Property 7: Deletion removes transaction from state and storage
fc.assert(fc.property(fc.array(validTransactionArb, { minLength: 1 }), (txs) => {
  const target = txs[0];
  const after = txs.filter(t => t.id !== target.id);
  const stored = JSON.parse(JSON.stringify(after));
  return !after.some(t => t.id === target.id)
    && !stored.some(t => t.id === target.id);
}), { numRuns: 100 });

// Feature: expense-budget-visualizer, Property 8: Out-of-range amounts are rejected
fc.assert(fc.property(
  fc.oneof(
    fc.double({ max: 0, noNaN: true }),                      // negative and zero
    fc.double({ min: 0.001, max: 0.009, noNaN: true }),      // below 0.01
    fc.double({ min: 1000000000, noNaN: true })               // above max
  ),
  (amount) => {
    const result = Validator.validate({ name: 'Test', amount: String(amount), category: 'Food' });
    return !result.valid && result.errors.amount !== undefined;
  }
), { numRuns: 100 });
```

### Integration / Smoke Tests (Manual)

Given the standalone HTML deployment, certain behaviors are best verified manually:

| Scenario | Verification |
|---|---|
| Page load restores saved transactions | Reload after adding items; confirm list/balance/chart match |
| Storage failure toast | Open in Safari private mode (or manually disable storage); confirm toast appears |
| Cross-browser rendering | Open in Chrome, Firefox, Edge, Safari latest; check layout at 320px and 1440px |
| Chart.js CDN failure | Throttle network to offline after page load; confirm app still functions |

### Accessibility Checks

- Validate ARIA live regions fire on toast/error messages using browser accessibility tree inspector
- Confirm focus management: form submit with errors should move focus to first invalid field
- Run WAVE or axe browser extension to catch contrast and label issues

> **Note**: Full WCAG 2.1 AA compliance validation requires manual testing with assistive technologies and expert accessibility review; automated tools catch approximately 30–40% of issues.
