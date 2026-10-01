# Implementation Plan: Expense & Budget Visualizer

## Overview

Implement a self-contained, single HTML file expense tracker using Vanilla JavaScript. The app is organized into six logical inline modules (`Storage`, `Validator`, `State`, `Render`, `Chart`, `App`) inside a single `<script type="module">` block, with Chart.js v4 loaded via CDN. All data persists in `localStorage`. The render pipeline is unidirectional: user event → state mutation → full re-render.

## Tasks

- [x] 1. Set up the HTML skeleton and global styles
  - Create `index.html` with the full document structure: `<header>` with `#balance-section`, `<main>` with `#form-section`, `#chart-section`, `#list-section`, and `#toast-container`
  - Add the Chart.js v4 CDN `<script>` tag before the module script
  - Write inline `<style>` with a CSS reset, typographic scale (minimum 14px body text), and responsive flexbox/grid layout that works from 320px to 2560px viewport widths
  - Define CSS custom properties for the three category colors: Food `#FF6384`, Transport `#36A2EB`, Fun `#FFCE56`
  - Style all interactive controls with visible `:hover` and `:focus` states
  - Style `.error-msg`, `.toast`, `.empty-state`, and `.category-badge` classes
  - Ensure contrast ratios meet WCAG 2.1 AA (minimum 4.5:1 for normal text)
  - _Requirements: 7.3, 8.3, 9.1, 9.2, 9.3, 9.4_

- [x] 2. Implement the `Storage` module
  - [x] 2.1 Write the `Storage` object with `KEY`, `load()`, and `save(transactions)` methods inside the `<script type="module">` block
    - `load()` wraps `localStorage.getItem` + `JSON.parse` in a try/catch; returns the parsed array or `null` on any error
    - `save(transactions)` wraps `localStorage.setItem` + `JSON.stringify` in a try/catch; returns `true` on success, `false` on failure
    - _Requirements: 6.1, 6.2, 6.4, 6.5_

  - [ ]* 2.2 Write property test for Storage round-trip (Property 4)
    - **Property 4: Storage round-trip preserves transaction data**
    - **Validates: Requirements 6.2, 6.3**
    - In `tests.html`, use `fast-check` via CDN; generate arbitrary valid transaction objects and assert that `JSON.parse(JSON.stringify([tx]))[0]` has identical `id`, `name`, `amount`, `category`, and `createdAt` fields

- [x] 3. Implement the `Validator` module
  - [x] 3.1 Write the `Validator` object with a `validate({ name, amount, category })` method
    - Name rule: trimmed length must be between 1 and 100; whitespace-only strings are treated as empty
    - Amount rule: parsed float must satisfy `0.01 ≤ value ≤ 999,999,999.99`
    - Category rule: value must be one of `'Food'`, `'Transport'`, `'Fun'`
    - Returns `{ valid: boolean, errors: { name?, amount?, category? } }`
    - _Requirements: 1.2, 1.3, 1.5_

  - [ ]* 3.2 Write property test for invalid item name rejection (Property 1)
    - **Property 1: Invalid item names are rejected**
    - **Validates: Requirements 1.2, 1.5**
    - Generate empty strings and whitespace-only strings; assert `Validator.validate()` returns `valid: false` with `errors.name` set

  - [ ]* 3.3 Write property test for out-of-range amount rejection (Property 8)
    - **Property 8: Out-of-range amounts are rejected**
    - **Validates: Requirements 1.1, 1.2, 1.3**
    - Generate values ≤ 0, values in (0, 0.01), and values > 999,999,999.99; assert `Validator.validate()` returns `valid: false` with `errors.amount` set

- [x] 4. Implement the `State` module
  - [x] 4.1 Write the `State` object with `transactions[]`, `addTransaction({ name, amount, category })`, `deleteTransaction(id)`, and `getAll()` methods
    - `addTransaction` creates a new transaction object with `id: crypto.randomUUID()`, trimmed `name`, parsed `amount`, `category`, and `createdAt: Date.now()`; pushes it to the array and returns the new transaction
    - `deleteTransaction(id)` filters the array in place by removing the matching `id`
    - `getAll()` returns a shallow copy of the array
    - _Requirements: 1.4, 2.3, 3.2_

  - [ ]* 4.2 Write property test for valid submission appending one transaction at top (Property 2)
    - **Property 2: Valid form submission appends one transaction at the top**
    - **Validates: Requirements 1.4, 2.3**
    - Generate valid input + an array of existing transactions; assert `addTransaction` increases length by 1 and the new transaction appears at index 0 when sorted by `createdAt` descending

  - [ ]* 4.3 Write property test for deletion removing transaction from state (Property 7)
    - **Property 7: Deletion removes transaction from state and storage**
    - **Validates: Requirements 3.2, 3.3**
    - Generate a non-empty array; pick the first transaction; call `deleteTransaction(id)` and assert neither the in-memory array nor a `JSON.parse(JSON.stringify(after))` contains the deleted `id`, and that `computeBalance` reflects the removal

- [x] 5. Checkpoint — core logic verified
  - Ensure all property tests in `tests.html` for Properties 1, 2, 4, 7, 8 pass before continuing. Ask the user if any issues arise.

- [x] 6. Implement balance computation and the `Render` module
  - [x] 6.1 Write the `computeBalance(transactions)` pure function that returns the arithmetic sum of all `amount` values (returns `0` for an empty array)
    - _Requirements: 4.1, 4.4_

  - [ ]* 6.2 Write property test for the balance invariant (Property 3)
    - **Property 3: Balance invariant — balance equals sum of all amounts**
    - **Validates: Requirements 4.1, 4.2, 4.3, 4.4**
    - Generate arbitrary arrays of valid transactions; assert `computeBalance(txs) === txs.reduce((s, t) => s + t.amount, 0)`

  - [x] 6.3 Write `renderTotalBalance(transactions)` — updates `#total-balance` `textContent` using `toLocaleString('en-US', { style: 'currency', currency: 'USD' })`
    - _Requirements: 4.1, 4.2, 4.3, 4.4_

  - [x] 6.4 Write `renderTransactionItem(tx)` — returns an HTML string (or `<li>` element) containing: item name (truncated at 100 chars via CSS `text-overflow: ellipsis`), amount formatted to 2 decimal places with `$` symbol, category badge, and a `<button class="delete-btn" aria-label="Delete {name}">✕</button>`
    - _Requirements: 2.1, 3.1_

  - [ ]* 6.5 Write property test for rendered item containing all required fields and delete control (Property 6)
    - **Property 6: Rendered transaction item contains all required fields and a delete control**
    - **Validates: Requirements 2.1, 3.1**
    - Generate arbitrary valid transaction objects; call `renderTransactionItem(tx)` and assert the output includes the name (up to 100 chars), a currency-formatted amount, the category label, and a `delete-btn` element

  - [x] 6.6 Write `renderTransactionList(transactions)` — renders the full `<ul id="transaction-list">` contents
    - Sort transactions by `createdAt` descending before rendering
    - Show `<li class="empty-state">No transactions yet.</li>` when the array is empty
    - _Requirements: 2.2, 2.3, 2.4, 2.5_

- [x] 7. Implement the `Chart` module
  - [x] 7.1 Write the `aggregateByCategory(transactions)` pure function
    - Returns `{ labels, data, colors }` where each index corresponds to one distinct category present in the transactions
    - Colors are mapped from the fixed palette: Food → `#FF6384`, Transport → `#36A2EB`, Fun → `#FFCE56`
    - _Requirements: 5.1, 5.3_

  - [ ]* 7.2 Write property test for chart data reflecting current transaction categories (Property 5)
    - **Property 5: Chart data reflects current transaction categories**
    - **Validates: Requirements 5.1, 5.2, 5.3, 5.4**
    - Generate a non-empty array of valid transactions; assert `aggregateByCategory` returns exactly one entry per distinct category and each entry's percentage equals `round(categoryTotal / grandTotal * 1000) / 10`

  - [x] 7.3 Write `initChart(ctx, transactions)` — creates the Chart.js pie chart instance once with `responsive: true`, legend at bottom, and tooltips enabled; stores it in a module-scoped variable
    - Handle CDN failure: if `Chart` is undefined, hide the canvas and show `<p>Chart unavailable — check your internet connection.</p>`
    - _Requirements: 5.1, 5.5, 7.2_

  - [x] 7.4 Write `updateChart(transactions)` — if the chart instance exists, calls `aggregateByCategory`, reassigns `chart.data`, and calls `chart.update()`; if no transactions, hides `<canvas>` and shows the `#chart-empty-msg` placeholder; otherwise shows the canvas
    - _Requirements: 5.2, 5.3, 5.5_

- [x] 8. Implement the `Toast` utility and error display helpers
  - [x] 8.1 Write `showToast(message, type = 'error')` — appends a `<div class="toast toast--{type}">` to `#toast-container` and removes it after 4 seconds via `setTimeout`; `#toast-container` must have `aria-live="polite"` and `aria-atomic="true"`
    - _Requirements: 6.4, 6.5, 3.4_

  - [x] 8.2 Write `renderErrors(errors)` — injects error text into each `.error-msg` span adjacent to the corresponding form field; clears any existing error messages before writing new ones; moves focus to the first invalid field
    - _Requirements: 1.3, 9.3_

- [x] 9. Implement the `App` module — bootstrap and event wiring
  - [x] 9.1 Write the app initialization function that runs on `DOMContentLoaded`
    - Call `Storage.load()`; if result is `null`, call `showToast('Saved data could not be loaded. Starting fresh.')` and use `[]`
    - Populate `State` with the loaded transactions
    - Initialize the chart via `initChart`
    - Call the full render pipeline: `renderTransactionList`, `renderTotalBalance`, `updateChart`
    - _Requirements: 6.2, 6.3, 6.4_

  - [x] 9.2 Wire the `#transaction-form` `submit` event handler
    - `e.preventDefault()`, read field values, call `Validator.validate()`
    - On failure: call `renderErrors(result.errors)` and return
    - On success: call `State.addTransaction()`, then `Storage.save(State.getAll())`; if save returns `false`, call `showToast('Your change could not be saved persistently.')`
    - Call the full render pipeline, then `form.reset()` and clear all `.error-msg` spans
    - _Requirements: 1.2, 1.3, 1.4, 1.5, 6.1, 6.5_

  - [x] 9.3 Wire the `#transaction-list` delegated `click` event handler for delete buttons
    - Use event delegation on `#transaction-list`; detect clicks on `.delete-btn` and read `data-id` from the closest `<li>`
    - Call `State.deleteTransaction(id)`, then `Storage.save(State.getAll())`; if save returns `false`, call `showToast('Your change could not be saved persistently.')` and do NOT remove the transaction from state
    - Call the full render pipeline
    - _Requirements: 3.2, 3.3, 3.4, 4.3_

- [x] 10. Checkpoint — full app integration
  - Open `index.html` directly in a browser (no server needed). Verify: adding transactions updates the list, balance, and chart; deleting a transaction removes it and updates all three displays; page reload restores all data. Ask the user if any issues arise.

- [x] 11. Create the property-based test harness
  - [x] 11.1 Create `tests.html` with fast-check v3 loaded via CDN and all 8 property tests from the design document implemented with `numRuns: 100` each
    - Export pure functions (`Validator.validate`, `computeBalance`, `aggregateByCategory`, `renderTransactionItem`, `addTransaction`, `deleteTransaction`) so they are accessible in the test harness
    - Annotate each test with a comment: `// Feature: expense-budget-visualizer, Property N: <title>`
    - _Requirements: 1.2, 1.5, 1.1, 1.3, 6.2, 6.3, 2.3, 3.2, 3.3, 4.1–4.4, 5.1–5.4, 2.1, 3.1_

  - [ ]* 11.2 Write unit tests for `Validator.validate()` covering boundary conditions
    - Blank name, whitespace-only name, amount = 0, amount = 0.005, amount = 999999999.99, amount = 1e9, valid form with all three categories
    - _Requirements: 1.1, 1.2, 1.3, 1.5_

  - [ ]* 11.3 Write unit tests for `Storage.load()` and `Storage.save()` covering error paths
    - Valid JSON, malformed JSON, missing key, simulated quota-exceeded exception
    - _Requirements: 6.1, 6.2, 6.4, 6.5_

  - [ ]* 11.4 Write unit tests for `aggregateByCategory()` covering edge cases
    - Empty list, single category, all three categories, deleting the last item in a category
    - _Requirements: 5.1, 5.3_

- [x] 12. Final checkpoint — all tests pass
  - Open `tests.html` in a browser and confirm all 8 property tests and all unit tests pass. Ensure all tests pass, ask the user if questions arise.

## Notes

- Tasks marked with `*` are optional and can be skipped for a faster MVP
- The entire app lives in `index.html`; tests live in `tests.html` (same directory) and import the pure-function modules by reference
- Pure functions (`computeBalance`, `aggregateByCategory`, `renderTransactionItem`, `Validator.validate`) should be written with no DOM side-effects so they are testable outside the browser context
- Each task references specific requirements for traceability
- Checkpoints (tasks 5, 10, 12) ensure incremental validation before moving on
- Property tests validate universal correctness properties; unit tests validate specific examples and edge cases

## Task Dependency Graph

```json
{
  "waves": [
    { "id": 0, "tasks": ["2.1", "3.1"] },
    { "id": 1, "tasks": ["4.1", "3.2", "3.3"] },
    { "id": 2, "tasks": ["4.2", "4.3", "6.1"] },
    { "id": 3, "tasks": ["2.2", "6.2", "6.3", "6.4", "7.1"] },
    { "id": 4, "tasks": ["6.5", "6.6", "7.2", "7.3", "8.1", "8.2"] },
    { "id": 5, "tasks": ["7.4", "9.1"] },
    { "id": 6, "tasks": ["9.2", "9.3"] },
    { "id": 7, "tasks": ["11.1", "11.2", "11.3", "11.4"] }
  ]
}
```
