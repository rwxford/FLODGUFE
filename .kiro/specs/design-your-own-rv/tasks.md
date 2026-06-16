# Implementation Plan: Design Your Own RV

## Overview

Build a fully client-side RV configurator page (`design.html`) using pure HTML, CSS, and vanilla JavaScript. The page allows visitors to select a Base Model and personalize it across 11 option categories, with a live price summary and quote form submission. All interactivity lives in a single `<script>` block at the bottom of `design.html`. Existing pages receive navigation updates and inventory cards receive "Design Your RV" links.

## Tasks

- [x] 1. Add "Design Your RV" navigation link to all existing pages
  - Add `<a href="design.html">Design Your RV</a>` to the `<nav>` in `index.html`, `inventory.html`, `drone-care.html`, and `about.html`, following the same link structure and ordering as the existing nav items
  - Add a "Design Your RV" call-to-action link to each inventory model card in `inventory.html` with `href="design.html?model=keowee"`, `href="design.html?model=blueridge"`, and `href="design.html?model=palmetto"` respectively, styled as `.cta-button`
  - _Requirements: 17.1, 17.4_

- [ ] 2. Create `design.html` shell with page structure, styles, and static markup
  - Create `design.html` with the standard site `<head>` (charset, viewport, title "Design Your Own RV – FLODGUFE RV", `styles.css` link)
  - Add a `<style>` block in `<head>` for all new scoped CSS — no changes to `styles.css`. Define classes: `.configurator-layout`, `.option-category`, `.option-card`, `.option-card--selected`, `.option-card__swatch`, `.price-sidebar`, `.price-footer`, `.design-summary`, `.design-summary__total`, `.form-error`, `#quote-confirmation`
  - Add the shared `<header>` with full nav including the new "Design Your RV" link marked as `class="active"`, and `<footer>`
  - Add a `.page-hero` section with heading "Design Your Own RV" and a brief subtitle
  - Add the `.configurator-layout` wrapper containing placeholder `<div id="configurator-main">` and `<div id="price-sidebar">`; add `<div id="price-footer">` as a sticky bottom bar for mobile
  - Add `<div id="model-selector">` and `<div id="option-categories">` and `<section id="design-summary">` inside `#configurator-main`
  - Add `<form id="quote-form">` with all required fields: First Name, Last Name, Email, Phone, Preferred Contact Method (radio: Email/Phone), Best Time to Contact (select), Additional Notes (textarea, maxlength="500"), hidden `<input id="design-summary-hidden">`, and submit button
  - Add `<div id="quote-confirmation" hidden>` confirmation message block
  - Add empty `<script>` block at the bottom of `<body>`
  - Implement responsive CSS: two-column grid layout (main + sidebar) at ≥ 768 px; single-column with sticky `#price-footer` at < 768 px, using existing CSS custom properties exclusively
  - _Requirements: 2.2, 14.1, 14.4, 17.2, 17.3, 18.3, 19.1, 19.2, 19.3, 19.4, 19.5, 20.4_

- [~] 3. Implement `OPTION_DATA` — embed all model and option data in the script block
  - Define `const MODELS` with `keowee` (basePrice: 139000), `blueridge` (basePrice: 159000), and `palmetto` (basePrice: 359000), each with `id`, `name`, `type`, `basePrice`, `description`, `image`, and `imageAlt`
  - Define `const OPTION_CATEGORIES` as an array of 11 category objects in the fixed order: `exterior-color`, `interior-color`, `flooring`, `kitchen`, `bathroom`, `bedroom`, `entertainment`, `climate`, `outdoor`, `safety`, `drone-care`
  - Populate each category with its full option list per requirements: Exterior Color (10 options, Req 4.1), Interior Color Scheme (10 options, Req 5.1), Flooring (4 options with maintenance notes, Req 6.1), Kitchen (3 options with included-items lists, Req 7.1), Bathroom (3 options with included-items lists, Req 8.1), Bedroom (3 options, Req 9.1), Entertainment & Technology (3 options, Req 10.1), Comfort & Climate (3 options with suitability notes, Req 11.1), Outdoor Living (3 options, Req 12.1), Safety & Convenience (3 options, Req 13.1), Drone Care Package (3 tiers per Req 21.1)
  - Each option object includes: `id`, `label`, `desc`, `upgradePrice`, `isDefault` (true for the first option), `swatch` (hex string for Exterior Color and Interior Color options), `excludedModels` array (empty for all current options), and any category-specific extra fields (`maintenanceNote`, `suitabilityNote`, `includes`)
  - _Requirements: 4.1, 5.1, 6.1, 7.1, 8.1, 9.1, 10.1, 11.1, 12.1, 13.1, 21.1, 20.3_

- [~] 4. Implement core state management and price calculation functions
  - Define `let state = { selectedModel: null, selections: {} }`
  - Implement `selectModel(modelId)` — sets `state.selectedModel`, resets `state.selections` to the `isDefault` option for every category in `OPTION_CATEGORIES`
  - Implement `selectOption(categoryId, optionId)` — updates `state.selections[categoryId]`
  - Implement `calculateTotal()` — returns `null` if no model selected; otherwise returns `MODELS[state.selectedModel].basePrice` plus the sum of `upgradePrice` for each currently selected option across all categories
  - _Requirements: 1.3, 1.4, 1.5, 3.4, 3.5, 14.2, 14.3_

- [ ]* 4.1 Write property test for `selectModel` (Property 1: model switch resets all selections to defaults)
  - **Property 1: Model Switch Resets All Selections to Defaults**
  - Use `fc.constantFrom('keowee','blueridge','palmetto')` for both initial and target model; `fc.record(...)` for arbitrary prior selections
  - After `selectModel(targetId)`: every key in `state.selections` must equal the `isDefault` option id for that category, and `calculateTotal()` must equal `MODELS[targetId].basePrice`
  - **Validates: Requirements 1.3, 1.4, 1.5**

- [ ]* 4.2 Write property test for `calculateTotal` (Property 5: total equals base price plus sum of upgrade prices)
  - **Property 5: Configured Total Equals Base Price Plus Sum of Upgrade Prices**
  - Use `fc.constantFrom('keowee','blueridge','palmetto')` for model; `fc.record(...)` mapping each category id to a valid option index
  - Assert `calculateTotal() === MODELS[modelId].basePrice + sum(selectedOption.upgradePrice for each category)` for all-defaults, all-maximum-upgrades, and arbitrary mixed selections
  - **Validates: Requirements 3.4, 3.5, 14.2, 14.3, 21.3**

- [~] 5. Implement model selector rendering and URL pre-selection
  - Implement `renderModelSelector()` — generates a `<fieldset>` with `<legend>Choose Your Model</legend>` and three `<label>`+`<input type="radio" name="base-model">` cards, each showing the model's exterior image, name (`<h3>`), starting price, and description; injects into `#model-selector`
  - Attach a `change` listener on the fieldset that calls `selectModel(modelId)` then `renderCategories()` then `updateSummary()`
  - On `DOMContentLoaded`, call `renderModelSelector()`, then read `?model=` from `location.search`; if the value is a valid key in `MODELS`, programmatically check the matching radio and call `selectModel()`, `renderCategories()`, `updateSummary()`; if absent or invalid, show no categories and display the "select a model" prompt
  - _Requirements: 1.1, 1.2, 1.3, 1.6, 17.4, 20.2_

- [~] 6. Implement `renderCategories(modelId)` — dynamic option category markup
  - Clear `#option-categories` and inject one `<fieldset class="option-category" id="cat-{categoryId}">` per category in `OPTION_CATEGORIES` order, each with `<legend>` containing a numbered span and the category label
  - For each option (filtered by `excludedModels`), inject a `<label class="option-card">` wrapping `<input type="radio">`, a swatch `<span>` (for color categories, with `style="background:{swatch}"` and `aria-label="{label} swatch"`), name span, description span, and price span (formatted as `+$0` or `+$X,XXX`)
  - For Flooring options, include a maintenance note element; for Comfort & Climate options, include a suitability note element; for Drone Care options, include the category-level note (Req 21.4) and, when the Elite tier radio is checked, render `<div id="drone-faa-notice">` with the FAA disclaimer (Req 21.5)
  - Check the radio corresponding to the current `state.selections[cat.id]` and add `.option-card--selected` to its parent `<label>`
  - Use event delegation: a single `change` listener on `#option-categories` calls `selectOption(categoryId, optionId)`, toggles `.option-card--selected`, and calls `updateSummary()`; for the drone-care category, also toggles `#drone-faa-notice` visibility
  - _Requirements: 2.1, 2.2, 3.1, 3.2, 3.3, 3.6, 4.2, 5.2, 6.3, 8.2, 11.2, 18.1, 18.2, 20.2, 21.2, 21.4, 21.5_

- [ ]* 6.1 Write property test for category order (Property 2: category order is fixed for every model)
  - **Property 2: Category Order Is Fixed for Every Model**
  - Use `fc.constantFrom('keowee','blueridge','palmetto')` for model
  - After `renderCategories(modelId)`, query all `.option-category` fieldset IDs from the DOM; assert they equal `['cat-exterior-color','cat-interior-color','cat-flooring','cat-kitchen','cat-bathroom','cat-bedroom','cat-entertainment','cat-climate','cat-outdoor','cat-safety','cat-drone-care']` in that exact order
  - **Validates: Requirements 2.1**

- [ ]* 6.2 Write property test for option rendering completeness (Property 3)
  - **Property 3: Option Rendering Completeness**
  - Use `fc.constantFrom('keowee','blueridge','palmetto')` for model; iterate all options after render
  - For each rendered option card, assert presence of: name text node, price text (matching `+$N` or `+$N,NNN`), description text node, swatch element with non-empty `aria-label` (color categories), maintenance note element (flooring category)
  - **Validates: Requirements 3.1, 4.2, 5.2, 6.3, 18.5**

- [ ]* 6.3 Write property test for exactly one selection per category (Property 4)
  - **Property 4: Exactly One Selection Per Category**
  - Use `fc.constantFrom('keowee','blueridge','palmetto')` for model; `fc.array(fc.nat())` for a sequence of option-index selections within a category
  - After each selection in the sequence: assert exactly one radio in the category is `checked`, and `.option-card--selected` appears on exactly one label; assert `state.selections[cat.id]` equals the last-selected option id
  - **Validates: Requirements 3.2, 3.3**

- [ ]* 6.4 Write property test for excluded options not rendered (Property 6)
  - **Property 6: Excluded Options Are Not Rendered for Their Excluded Models**
  - Use `fc.constantFrom('keowee','blueridge','palmetto')` for model; iterate options with non-empty `excludedModels`
  - After `renderCategories(modelId)`, assert no DOM element exists for any option whose `excludedModels` includes `modelId`
  - **Validates: Requirements 3.6, 8.2**

- [ ]* 6.5 Write property test for semantic HTML structure (Property 11)
  - **Property 11: Semantic HTML Structure for Every Rendered Category**
  - Use `fc.constantFrom('keowee','blueridge','palmetto')` for model
  - After `renderCategories(modelId)`, for every `.option-category` element: assert it is a `<fieldset>`, has a `<legend>` as a direct child, and all interactive controls are `<input type="radio">` elements inside `<label>` elements
  - **Validates: Requirements 18.1**

- [~] 7. Checkpoint — verify core rendering pipeline
  - Ensure all tests pass, ask the user if questions arise.

- [~] 8. Implement `updateSummary()` — live price summary and design summary
  - Implement `updateSummary()` to recalculate via `calculateTotal()` and update both `#price-sidebar` and `#price-footer` with the current model name, itemised list (one row per category showing selected option label and formatted upgrade price), and total
  - When `state.selectedModel` is `null`, render "Select a model to begin" in both summary nodes and no price; otherwise render the full itemised breakdown
  - Ensure `#price-sidebar`'s itemised list element has `aria-live="polite"` and `aria-atomic="true"` (set in static HTML); `updateSummary()` only mutates `textContent`/`innerHTML` of the list
  - Render `#design-summary` as a table or definition list with one row per category (category label + selected option name + upgrade price), each row containing an `<a href="#cat-{categoryId}">Edit</a>` link; display total price prominently with class `design-summary__total`
  - _Requirements: 14.1, 14.2, 14.3, 14.4, 14.5, 15.1, 15.2, 15.3, 15.4, 18.3, 21.3, 21.6_

- [ ]* 8.1 Write property test for design summary reflecting all selections (Property 7)
  - **Property 7: Design Summary Reflects All Current Selections**
  - Use `fc.constantFrom('keowee','blueridge','palmetto')` for model; `fc.record(...)` for arbitrary option selections
  - After `updateSummary()`: assert `#price-sidebar` and `#design-summary` each contain exactly 11 category entries, each displaying the selected option's label and upgrade price, and the total equals `calculateTotal()`
  - **Validates: Requirements 14.2, 15.1, 15.3**

- [~] 9. Implement form validation and quote form submission
  - Implement `validateEmail(input)` — returns `false` for strings missing `@`, missing domain, or with leading/trailing whitespace; returns `true` for valid email format
  - Implement `validateForm()` — checks First Name, Last Name, Email (using `validateEmail`), Preferred Contact Method, and Best Time to Contact; returns an error map `{ fieldId: errorMessage }` (empty map means valid)
  - Implement `renderErrors(errorMap)` — for each field in the error map, inject/update `<span role="alert" class="form-error" id="err-{fieldId}">` adjacent to the field; set `aria-describedby="err-{fieldId}"` and `aria-invalid="true"` on the input; clear error spans for valid fields and remove those attributes
  - Implement `serialiseSummary(state)` — returns a plain-text string with model name and base price, one line per category (label + selected option label + formatted upgrade price), and the configured total; returns empty string if no model selected
  - Attach a `submit` listener on `#quote-form` that calls `e.preventDefault()`, then `validateForm()`; if errors, call `renderErrors(errorMap)` and stop; if valid, call `serialiseSummary(state)`, set `document.getElementById('design-summary-hidden').value`, hide `#quote-form`, show `#quote-confirmation`, and call `form.submit()`
  - _Requirements: 16.1, 16.2, 16.3, 16.4, 16.5, 16.6, 16.7, 16.8_

- [ ]* 9.1 Write property test for form validation rejecting missing required fields (Property 8)
  - **Property 8: Form Validation Rejects Any Submission Missing a Required Field**
  - Use `fc.record(...)` for form field values; `fc.subarray(['firstName','lastName','email','contactMethod','bestTime'])` to choose which required fields are absent
  - For every combination where at least one required field is missing or empty: assert `validateForm()` returns a non-empty error map with an entry for each missing field
  - **Validates: Requirements 16.3, 16.4**

- [ ]* 9.2 Write property test for email validation (Property 9)
  - **Property 9: Email Validation Rejects Invalid Formats and Accepts Valid Ones**
  - Use `fc.emailAddress()` for valid inputs — assert `validateEmail(input)` returns `true`
  - Use `fc.string()` filtered to strings clearly missing `@` or domain — assert `validateEmail(input)` returns `false`
  - **Validates: Requirements 16.5**

- [ ]* 9.3 Write property test for design summary serialisation (Property 10)
  - **Property 10: Design Summary Serialisation Contains All Required Information**
  - Use `fc.constantFrom('keowee','blueridge','palmetto')` for model; `fc.record(...)` for arbitrary selections
  - Assert `serialiseSummary(state)` contains: model name, base price, all 11 category labels with their selected option labels and upgrade prices, and the configured total — no category omitted
  - **Validates: Requirements 16.8**

- [~] 10. Checkpoint — verify form validation and submission pipeline
  - Ensure all tests pass, ask the user if questions arise.

- [~] 11. Set up Jest + jsdom + fast-check test environment and write unit tests
  - Initialise `package.json` with `npm init -y` (or create it manually), add `jest`, `jest-environment-jsdom`, and `fast-check` as dev dependencies in `package.json` with pinned versions; add `"test": "jest --testEnvironment jsdom"` script
  - Create `jest.config.js` (or add `"jest"` config key in `package.json`) setting `testEnvironment: 'jsdom'`
  - Create `tests/configurator-logic.js` by extracting all pure functions from the `<script>` block (`MODELS`, `OPTION_CATEGORIES`, `calculateTotal`, `validateEmail`, `validateForm`, `serialiseSummary`, `selectModel`, `selectOption`, `renderCategories`, `updateSummary`) so they can be `require`d by Jest without a browser
  - Create `tests/configurator.test.js` and write unit tests for:
    - Data completeness: `OPTION_CATEGORIES.length === 11`; each category has the exact option count and prices from requirements
    - Initial state: `calculateTotal()` returns `null` before model selection
    - URL pre-selection: `selectModel('keowee')` called for `?model=keowee`; no selection for `?model=invalid`
    - Elite Drone Care FAA notice: selecting `drone-elite` renders `#drone-faa-notice` in DOM
    - ARIA live region: `#price-sidebar` has `aria-live="polite"` and `aria-atomic="true"`
    - Edit links: each row in `#design-summary` contains an `<a>` whose `href` matches `#cat-{categoryId}`
    - Form confirmation: after `handleSubmit` with all valid inputs, `#quote-confirmation` is visible and `#quote-form` is hidden
  - _Requirements: 4.1, 5.1, 6.1, 7.1, 8.1, 9.1, 10.1, 11.1, 12.1, 13.1, 14.5, 15.4, 16.6, 18.3, 21.5_

- [~] 12. Final checkpoint — full test suite and integration review
  - Ensure all unit tests and property-based tests pass with `npm test`
  - Verify that `design.html` loads without JS errors in a browser, all four existing pages have the "Design Your RV" nav link, and the inventory cards link to `design.html?model=*`
  - Ensure all tests pass, ask the user if questions arise.

## Notes

- Tasks marked with `*` are optional and can be skipped for a faster MVP build
- Each task references specific requirements for traceability
- All JavaScript must remain in the single `<script>` block at the bottom of `design.html`; `tests/configurator-logic.js` is a test-only extraction of the same logic
- Property tests validate universal correctness properties across all input combinations; unit tests cover specific scenarios and edge cases
- Checkpoints at tasks 7 and 10 validate incremental progress before moving to the next phase
- The `mailto:` form action is consistent with the existing site pattern; a comment in the HTML notes it can be replaced with a server-side endpoint
