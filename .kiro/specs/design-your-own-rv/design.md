# Design Document: Design Your Own RV

## Overview

The "Design Your Own RV" feature adds an interactive, fully client-side configuration page (`design.html`) to the FLODGUFE RV static website. Visitors select one of three luxury Base Models and personalize it across 11 Option Categories (exterior color, interior scheme, flooring, kitchen, bathroom, bedroom, entertainment, climate, outdoor living, safety, and drone care). A live price summary updates in real time. When satisfied, the visitor submits a quote request via an HTML form that embeds their complete Design Summary.

**Core constraints:**
- Pure static site: HTML + CSS + vanilla JavaScript only — no frameworks, no build tools.
- All option data embedded in the page at load time; zero network requests after initial load.
- All JavaScript in a single `<script>` block at the bottom of `design.html`.
- Must reuse `styles.css` custom properties and conventions; no new global CSS file.
- Responsive: sidebar layout ≥ 768 px; sticky-footer summary < 768 px.
- All interactive responses ≤ 500 ms (synchronous DOM operations — trivially met).

---

## Architecture

### File Structure

```
/                        ← existing site root
├── styles.css           ← existing (no modifications to global rules)
├── design.html          ← NEW page
├── index.html           ← nav updated: add "Design Your RV" link
├── inventory.html       ← nav + model cards updated
├── drone-care.html      ← nav updated
├── about.html           ← nav updated
└── images/              ← existing images (no new images required beyond swatches)
```

All interactivity lives inside a single `<script>` block at the bottom of `design.html`. No external JS files, no CDN imports, no module bundling.

### Runtime Data Flow

```
Page Load
  └─ OPTION_DATA constant (embedded JS object) initialised
  └─ DOM rendered with model-selector cards and empty configurator shell

User selects Base Model
  └─ renderCategories(modelId)      → injects fieldset/legend/label/radio markup
  └─ updateSummary()                → recalculates and paints Live Price Summary

User changes any radio
  └─ onChange handler fires (≤ 1 ms synchronous)
  └─ updateSummary()                → recalculates total, updates ARIA live region

User submits Quote Form
  └─ validateForm()                 → returns error map
  └─ if errors → renderErrors()    → inline messages, no submission
  └─ if valid  → injectHiddenField()→ serialise Design Summary into hidden input
                └─ form.submit()    → mailto or form-handler POST
```

### Page Layout (two-column ≥ 768 px)

```
┌─────────────────────────────────────────────────────────┐
│  Header (shared nav, sticky)                            │
├───────────────────────────────────┬─────────────────────┤
│  .configurator-main               │  .price-sidebar     │
│  ┌─────────────────────────────┐  │  (sticky top: 80px) │
│  │ #model-selector             │  │  Base Model         │
│  │ (3 radio cards)             │  │  + itemised options │
│  ├─────────────────────────────┤  │  ──────────────     │
│  │ #option-categories          │  │  TOTAL: $XXX,XXX    │
│  │ (fieldsets, one per cat.)   │  │                     │
│  ├─────────────────────────────┤  └─────────────────────┘
│  │ #design-summary             │
│  │ (review table + edit links) │
│  ├─────────────────────────────┤
│  │ #quote-form                 │
│  └─────────────────────────────┘
├─────────────────────────────────────────────────────────┤
│  Footer                                                 │
└─────────────────────────────────────────────────────────┘
```

On viewports < 768 px the sidebar becomes a `position: sticky; bottom: 0` footer bar that shows only the total. The full itemised summary is visible in the `#design-summary` section.

---

## Components and Interfaces

### 1. Model Selector (`#model-selector`)

A `<fieldset>` with `<legend>Choose Your Model</legend>` containing three `<label>` + `<input type="radio" name="base-model">` pairs styled as clickable cards.

Each card displays:
- Exterior thumbnail image (`images/{slug}_exterior.png`)
- Model name (`<h3>`)
- Starting price
- Short description (2–3 lines)

On `change`, the selector calls `renderCategories(modelId)` and `updateSummary()`.

URL pre-selection: on `DOMContentLoaded`, read `?model=keowee|blueridge|palmetto` from `location.search` and programmatically check the matching radio, then trigger `renderCategories`.

### 2. Option Categories (`#option-categories`)

Rendered dynamically by `renderCategories(modelId)`. Produces one `<fieldset>` per category, each containing:

```html
<fieldset class="option-category" id="cat-exterior-color">
  <legend class="option-category__legend">
    <span class="option-category__number">1</span>
    Exterior Color
  </legend>
  <!-- one label+radio per option -->
  <label class="option-card [option-card--selected]">
    <input type="radio" name="exterior-color" value="classic-white" checked>
    <span class="option-card__swatch" style="background:#ffffff" aria-label="Classic White swatch"></span>
    <span class="option-card__name">Classic White</span>
    <span class="option-card__desc">Standard factory finish</span>
    <span class="option-card__price">+$0</span>
  </label>
  …
</fieldset>
```

Event delegation: a single `change` listener on `#option-categories` handles all radio changes, calls `updateSummary()` and toggles the `.option-card--selected` class.

### 3. Live Price Summary (`#price-sidebar` / `.price-footer`)

Two DOM nodes exist in the HTML:
- `#price-sidebar` — visible at ≥ 768 px (`display: block` → `display: none` at < 768 px via media query).
- `#price-footer` — sticky footer, visible at < 768 px, shows only total.

Both have `role="region"` and `aria-label="Live Price Summary"`. The itemised list inside `#price-sidebar` has `aria-live="polite"` and `aria-atomic="true"` so screen readers announce the updated total after every change.

`updateSummary()` renders:
```
[Model Name] ………………… $XXX,000
  Exterior Color — Classic White … +$0
  Interior Color — Coastal Cream … +$0
  …
  Drone Care Package — Standard … +$0
─────────────────────────────────
Configured Total: $XXX,XXX
```

### 4. Design Summary Section (`#design-summary`)

A static section below `#option-categories`, updated by `updateSummary()`. Renders a `<table>` or definition list of category → selection. Each row has an "Edit" `<a>` that fires `scrollToCategory(categoryId)` (smooth scroll with `scrollIntoView({behavior:'smooth'})`).

Displays total price prominently with `class="design-summary__total"`.

### 5. Quote Form (`#quote-form`)

Standard HTML `<form>` using existing `.quote-form`, `.form-row`, `.form-group` classes.

Fields:
| Field | Type | Required |
|---|---|---|
| First Name | `text` | ✓ |
| Last Name | `text` | ✓ |
| Email Address | `email` | ✓ |
| Phone Number | `tel` | — |
| Preferred Contact Method | `radio` (Email / Phone) | ✓ |
| Best Time to Contact | `select` | ✓ |
| Additional Notes | `textarea` (max 500 chars) | — |
| Design Summary | `hidden` (auto-populated) | — |

Client-side validation fires on `submit` event (prevented with `e.preventDefault()` until valid). Inline error messages injected as `<span role="alert" class="form-error">` adjacent to each invalid field.

On successful validation:
1. `document.getElementById('design-summary-hidden').value` is set to the serialised Design Summary string.
2. Confirmation message shown (form hidden, `<div id="quote-confirmation">` shown).
3. `form.submit()` called.

`action="mailto:sales@flodguferv.com"` with `method="post" enctype="text/plain"` (consistent with existing site pattern). A comment in the HTML notes this can be replaced with a server-side endpoint.

---

## Data Models

All option data is defined in a single `const OPTION_DATA` object embedded in the `<script>` block. No data files, no fetch calls.

### Base Models

```js
const MODELS = {
  keowee: {
    id: 'keowee',
    name: 'Keowee Royale',
    type: 'Fifth Wheel',
    basePrice: 139000,
    description: 'Designed for easy living…',
    image: 'images/keowee_exterior.png',
    imageAlt: 'FLODGUFE Keowee Royale exterior'
  },
  blueridge: { … basePrice: 159000 … },
  palmetto:  { … basePrice: 359000 … }
};
```

### Option Category Structure

```js
const OPTION_CATEGORIES = [
  {
    id: 'exterior-color',
    label: 'Exterior Color',
    options: [
      {
        id: 'classic-white',
        label: 'Classic White',
        desc: 'Standard factory finish, brilliant gloss white.',
        upgradePrice: 0,
        swatch: '#FFFFFF',   // CSS color for inline swatch
        isDefault: true
      },
      {
        id: 'midnight-navy',
        label: 'Midnight Navy',
        desc: 'Deep navy blue full-body wrap.',
        upgradePrice: 1500,
        swatch: '#003366'
      },
      // … 8 more options
    ]
  },
  {
    id: 'interior-color',
    label: 'Interior Color Scheme',
    options: [ /* 10 options per requirements */ ]
  },
  {
    id: 'flooring',
    label: 'Flooring',
    options: [
      {
        id: 'vinyl-plank',
        label: 'Residential Vinyl Plank',
        desc: 'Easy clean, no refinishing required.',
        upgradePrice: 0,
        isDefault: true
      },
      // … 3 more
    ]
  },
  // … 8 more categories (kitchen, bathroom, bedroom, entertainment,
  //   climate, outdoor, safety, drone-care)
];
```

### Category-Option Compatibility

Requirement 8.2 notes that the Luxury Spa Bathroom Package requires a minimum model length. Rather than a complex compatibility matrix (all three models support all packages per the requirements as written), a simple `excludedModels: ['keowee']` array on an option object gates display:

```js
{
  id: 'luxury-spa',
  label: 'Luxury Spa Package',
  upgradePrice: 6500,
  excludedModels: []   // available on all models per current requirements
}
```

`renderCategories(modelId)` filters options with `option.excludedModels?.includes(modelId)` before rendering.

### Application State

A single plain object tracks current selections:

```js
let state = {
  selectedModel: null,          // 'keowee' | 'blueridge' | 'palmetto' | null
  selections: {
    // keyed by OPTION_CATEGORIES[i].id → option id string
    // e.g. { 'exterior-color': 'classic-white', 'flooring': 'vinyl-plank', … }
  }
};
```

State is mutated only by:
- `selectModel(modelId)` — sets `state.selectedModel`, resets `state.selections` to defaults.
- `selectOption(categoryId, optionId)` — updates single key in `state.selections`.

Both functions call `updateSummary()` after mutation.

### Price Calculation

```js
function calculateTotal() {
  if (!state.selectedModel) return null;
  const base = MODELS[state.selectedModel].basePrice;
  const upgrades = OPTION_CATEGORIES.reduce((sum, cat) => {
    const selectedId = state.selections[cat.id];
    const opt = cat.options.find(o => o.id === selectedId);
    return sum + (opt?.upgradePrice ?? 0);
  }, 0);
  return base + upgrades;
}
```

### Design Summary Serialisation

For the hidden form field, a plain-text serialisation:

```
Model: Keowee Royale ($139,000)
Exterior Color: Classic White (+$0)
Interior Color Scheme: Coastal Cream (+$0)
Flooring: Residential Vinyl Plank (+$0)
Kitchen Package: Gourmet Package (+$4,500)
Bathroom Package: Standard Package (+$0)
Bedroom Package: Resort Package (+$5,500)
Entertainment & Technology: Connected Traveler Package (+$3,100)
Comfort & Climate: Standard Package (+$0)
Outdoor Living: Standard Package (+$0)
Safety & Convenience: Enhanced Safety Package (+$2,200)
Drone Care Package: Advanced Drone Care Package (+$2,400/year)
---
Configured Total: $156,700
```

### CSS Additions (inline in `design.html` `<style>` block)

All new styles are scoped to `design.html` via a `<style>` block in `<head>` — no modifications to the shared `styles.css`. They exclusively use the existing CSS custom properties (`--color-navy`, `--color-gold`, `--font-body`, etc.) and follow existing naming conventions.

Key new classes:

| Class | Purpose |
|---|---|
| `.configurator-layout` | CSS Grid wrapper: `[main] [sidebar]` at ≥ 768 px |
| `.option-category` | Fieldset styling with gold-bordered legend |
| `.option-card` | Radio-option card with hover/focus/selected states |
| `.option-card--selected` | Gold border + light navy background for active selection |
| `.option-card__swatch` | 32 × 32 px inline color swatch circle |
| `.price-sidebar` | Sticky sidebar at ≥ 768 px |
| `.price-footer` | Sticky bottom bar at < 768 px |
| `.design-summary` | Summary table section |
| `.design-summary__total` | Large gold-highlighted total price |
| `.form-error` | Red inline validation error text |
| `#quote-confirmation` | Hidden success message, shown after submit |

---

## Correctness Properties

*A property is a characteristic or behavior that should hold true across all valid executions of a system — essentially, a formal statement about what the system should do. Properties serve as the bridge between human-readable specifications and machine-verifiable correctness guarantees.*

---

### Property 1: Model Switch Resets All Selections to Defaults

*For any* initial selection state (any previously chosen options across all categories) and any target Base Model, calling `selectModel(modelId)` must result in `state.selections` containing exactly the default option for every Option Category, and `calculateTotal()` must equal the target model's `basePrice` (since all default upgrade prices are `+$0`).

**Validates: Requirements 1.3, 1.4, 1.5**

---

### Property 2: Category Order Is Fixed for Every Model

*For any* Base Model selection, the ordered list of rendered `<fieldset>` category IDs in the DOM must exactly equal `['exterior-color', 'interior-color', 'flooring', 'kitchen', 'bathroom', 'bedroom', 'entertainment', 'climate', 'outdoor', 'safety', 'drone-care']` — regardless of which model was selected or any prior state.

**Validates: Requirements 2.1**

---

### Property 3: Option Rendering Completeness

*For any* rendered Option Category and *for any* option within that category (as defined in `OPTION_DATA`), the rendered DOM element for that option must contain: (a) the option name text, (b) the upgrade price text (formatted as `+$X,XXX` or `+$0`), (c) a description text node, (d) a color swatch element with non-empty `aria-label` for options in the Exterior Color and Interior Color Scheme categories, and (e) a maintenance note element for options in the Flooring category.

**Validates: Requirements 3.1, 4.2, 5.2, 6.3, 18.5**

---

### Property 4: Exactly One Selection Per Category

*For any* Option Category and *for any* sequence of option selections within that category, at all times exactly one option in the category is in the selected state (both in `state.selections` and as a checked radio in the DOM). Selecting a new option within a category must deselect the previously selected option.

**Validates: Requirements 3.2, 3.3**

---

### Property 5: Configured Total Equals Base Price Plus Sum of Upgrade Prices

*For any* Base Model and *for any* combination of one option per Option Category, `calculateTotal()` must equal `MODELS[modelId].basePrice + sum(selectedOption.upgradePrice for each category)`. This must hold for all possible combinations of selections including all-defaults, all-maximum-upgrades, and arbitrary mixed selections.

**Validates: Requirements 3.4, 3.5, 14.2, 14.3, 21.3**

---

### Property 6: Excluded Options Are Not Rendered for Their Excluded Models

*For any* option in `OPTION_DATA` that has a non-empty `excludedModels` array, and *for any* model ID listed in that array, calling `renderCategories(modelId)` must not produce a DOM element for that option. The option should only appear when a non-excluded model is selected.

**Validates: Requirements 3.6, 8.2**

---

### Property 7: Design Summary Reflects All Current Selections

*For any* model selection and *for any* combination of option selections, the Design Summary section (both `#price-sidebar` and `#design-summary`) must contain one entry per Option Category displaying the currently selected option's name and upgrade price, plus the `calculateTotal()` result. The number of entries must always equal the total number of Option Categories (11).

**Validates: Requirements 14.2, 15.1, 15.3**

---

### Property 8: Form Validation Rejects Any Submission Missing a Required Field

*For any* form input state where at least one required field (First Name, Last Name, Email, Preferred Contact Method, Best Time to Contact) is empty or absent, `validateForm()` must return a non-empty error map containing an entry for the missing field, and the form must not be submitted. This must hold for every combination of missing required fields.

**Validates: Requirements 16.3, 16.4**

---

### Property 9: Email Validation Rejects Invalid Formats and Accepts Valid Ones

*For any* string that does not conform to the standard email address format (e.g., missing `@`, missing domain, leading/trailing whitespace), `validateEmail(input)` must return `false`. *For any* string that is a valid email address (contains exactly one `@`, non-empty local part, non-empty domain with at least one `.`), `validateEmail(input)` must return `true`.

**Validates: Requirements 16.5**

---

### Property 10: Design Summary Serialisation Contains All Required Information

*For any* model selection and *for any* combination of option selections, `serialiseSummary(state)` must return a string that contains: (a) the Base Model name, (b) the Base Model starting price, (c) for every Option Category, the category label and selected option label, (d) every selected option's upgrade price, and (e) the configured total. No selection may be omitted from the serialised output.

**Validates: Requirements 16.8**

---

### Property 11: Semantic HTML Structure for Every Rendered Category

*For any* Base Model selection, every rendered Option Category must be wrapped in a `<fieldset>` element, have a `<legend>` element as a direct child, and contain only `<label>` elements wrapping `<input type="radio">` elements for the individual options — never bare text or `<div>`-only structures for interactive controls.

**Validates: Requirements 18.1**

---

## Error Handling

### Missing or Invalid URL Query Parameter

- `?model=` absent or unrecognised value → no model pre-selected; user sees model selector prompt; no JS error thrown.
- Handled by: `const model = new URLSearchParams(location.search).get('model'); if (MODELS[model]) selectModel(model);`

### Empty Option Data (defensive)

- If `OPTION_CATEGORIES` is empty or a category has no options, `renderCategories` skips that category silently. A `console.warn` is emitted in development.

### Form Submission Errors

- If `mailto:` is blocked by the browser/OS, the form does not silently fail — the browser's default `mailto:` fallback behaviour applies (error dialog from OS mail client). A comment in the HTML notes this and suggests replacing with a server-side endpoint.

### State Corruption

- No async operations, no external data, no storage reads — state cannot be corrupted by race conditions or stale data.

### Accessibility Error States

- Inline form errors set `aria-describedby` on the offending input pointing to the error `<span>` ID, and `aria-invalid="true"` on the input, so screen readers announce both the field label and the error on focus.

---

## Testing Strategy

### PBT Applicability Assessment

This feature is a client-side UI configurator built in vanilla JavaScript. The core logic — price calculation, state management, form validation, option rendering, and summary serialisation — consists of **pure or near-pure functions** with clear input/output behaviour and large input spaces. Property-based testing is appropriate for these logic layers.

UI rendering and CSS layout are **not** suitable for PBT. Those are covered by snapshot tests and manual/visual review.

### Test Environment

Since there is no build tool or module system, tests will use **Jest** with **jsdom** (the standard choice for testing vanilla DOM logic without a browser). The property-based testing library is **fast-check** (the leading JS PBT library).

Install (for development only — not deployed):
```
npm init -y
npm install --save-dev jest jest-environment-jsdom fast-check
```

Tests live in `tests/configurator.test.js`. Source logic is extracted from the `<script>` block into a testable module by copying pure functions (no DOM dependencies) into `tests/configurator-logic.js` which can be `require`d by Jest.

### Unit Tests (example-based)

These cover specific concrete scenarios and edge cases:

1. **Data completeness** — `OPTION_DATA` contains exactly 11 categories; Exterior Color has exactly 10 options with the prices specified in Requirements 4.1; Interior Color has 10 options (Req 5.1); Flooring has 4 options (Req 6.1); Kitchen has 3 (Req 7.1); Bathroom has 3 (Req 8.1); etc.
2. **Initial state** — before any model selected, `calculateTotal()` returns `null`; summary shows "Select a model to begin".
3. **URL pre-selection** — `?model=keowee` triggers `selectModel('keowee')` on load; `?model=invalid` triggers no selection.
4. **Elite Drone Care FAA disclaimer** — selecting `drone-elite` renders a `#drone-faa-notice` element in the DOM.
5. **Navigation links** — `design.html` nav contains active-page class; all four other pages contain "Design Your RV" nav link.
6. **Inventory cards** — each inventory card's "Design Your RV" link href includes `?model=` param.
7. **ARIA live region** — `#price-sidebar` has `aria-live="polite"` and `aria-atomic="true"` attributes.
8. **Edit links in Design Summary** — each category row in `#design-summary` contains an `<a>` pointing to `#cat-{categoryId}`.
9. **Form confirmation** — after `handleSubmit` with all valid inputs, `#quote-confirmation` is visible and `#quote-form` is hidden.

### Property-Based Tests (fast-check)

Each test is configured to run a minimum of **100 iterations**. Tag format in comments: `Feature: design-your-own-rv, Property N: <property text>`.

| Test | Property | fast-check arbitraries |
|---|---|---|
| PBT-1 | Model switch resets all selections to defaults | `fc.constantFrom('keowee','blueridge','palmetto')` for initial and target model; `fc.record(...)` for random prior selections |
| PBT-2 | Category order is fixed for every model | `fc.constantFrom(...)` for model; verify rendered order |
| PBT-3 | Option rendering completeness | `fc.constantFrom(...)` for model; iterate all options after render |
| PBT-4 | Exactly one selection per category | `fc.constantFrom(...)` for model; `fc.array(fc.nat())` for sequence of selection indices |
| PBT-5 | Total = base + sum of upgrades | `fc.constantFrom(...)` for model; `fc.record(...)` mapping each category to a valid option index |
| PBT-6 | Excluded options not rendered | `fc.constantFrom(...)` for model; verify absent options |
| PBT-7 | Design Summary reflects all selections | `fc.constantFrom(...)` for model; `fc.record(...)` for selections |
| PBT-8 | Validation rejects missing required fields | `fc.record(...)` for form fields; `fc.boolean()` for which required fields are absent |
| PBT-9 | Email validation (valid/invalid) | `fc.emailAddress()` for valid; `fc.string()` filtered for clearly invalid formats |
| PBT-10 | Serialisation contains all required information | `fc.constantFrom(...)` for model; `fc.record(...)` for selections |
| PBT-11 | Semantic HTML for every category | `fc.constantFrom(...)` for model; verify DOM structure |

### Manual / Visual Testing

- Responsive layout at 320 px, 768 px, 1024 px (browser DevTools).
- Keyboard navigation: Tab through all controls, Enter/Space to select, focus ring visible.
- Screen reader: VoiceOver/NVDA announces live region update after each option change.
- Color contrast: verify gold-on-navy and navy-on-white pass 4.5:1 with a browser contrast checker.
- WCAG full validation requires manual testing with assistive technologies and expert accessibility review.
