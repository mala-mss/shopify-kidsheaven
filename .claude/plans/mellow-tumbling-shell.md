# Fix non-functioning sort menu in kidsheaven theme

## Context

The sort menu (the "Sort by" dropdown) on collection/search pages is broken in two ways:

1. **Desktop listbox (radio buttons) is completely dead.** `snippets/sorting.liquid:132` wires `on:change="/selectFromRow"` on each radio input, but no `selectFromRow` method exists on `SortingFilterComponent` in `assets/facets.js`. The change event fires, the `Component` event-delegation code in `assets/component.js` (lines 248–275) tries to invoke `instance['selectFromRow']`, gets `undefined`, the `typeof callback === 'function'` check fails, and **nothing happens** — the form is never submitted, the URL never updates, the page never re-sorts. The browser also throws a `ReferenceError` for any consumers that log it.

2. **The summary status label ("Sort by: Featured") never updates on desktop** because `SortingFilterComponent#updateFacetStatus` in `assets/facets.js:681-692` bails out unless `event.target instanceof HTMLSelectElement`. On desktop, the change event target is a radio `HTMLInputElement`, so the early-return prevents the status from being refreshed — even after the form *does* submit (e.g. once #1 is fixed).

3. **The mobile sorting-only form never receives its radio value** in the submission. The form in `blocks/filters.liquid:339` has `id="FacetFiltersForm--{{ section.id }}-mobile-sorting-only"`, but the radio inputs in `snippets/sorting.liquid:124` use `form="FacetFiltersForm--{{ section_id }}-{{ suffix }}"` where `suffix` is `"mobile"`. The id doesn't match, so the browser drops the radio value on submit, and the URL ends up with no `sort_by` parameter.

After all three are fixed, the user can pick any sort option in any viewport and the page will re-render with the new `?sort_by=…` URL.

## Files to modify

- `snippets/sorting.liquid` — only file that needs a Liquid change
- `assets/facets.js` — only file that needs a JS change
- `assets/component.js` — **no changes** (read-only reference for how `on:change` is dispatched)

## Changes

### 1. `snippets/sorting.liquid` — fix `selectFromRow` and form id

**A. Replace the `on:change="/selectFromRow"` binding** (line 132) with the correct handler that already exists:

```diff
-                on:change="/selectFromRow"
+                on:change="/updateFilterAndSorting"
```

`SortingFilterComponent#updateFilterAndSorting` (facets.js:628) is the existing handler for sort changes — it locates the `facets-form-component`, calls `facetsForm.updateFilters()`, and closes the `<details>` dropdown. It already handles both the SELECT (mobile) and radio (desktop) inputs via its `shouldDisable` logic. Repurposing it for the radio change event is the minimal fix and matches what the rest of the component already expects.

**B. Fix the radio `form=` attribute for mobile sorting-only** (lines 122-125). When `suffix == 'mobile'`, we need the form id to be `FacetFiltersForm--{{ section_id }}-mobile-sorting-only` (the form defined in `blocks/filters.liquid:339`), not `FacetFiltersForm--{{ section_id }}-mobile`. The cleanest place to fix this is in `snippets/sorting.liquid` since the snippet is the one rendering the radio inputs:

```diff
-              <input
-                type="radio"
-                name="sort_by"
-                id="sort-option-{{ option.value }}-{{ section_id }}"
-                {% if suffix != blank %}
-                  form="FacetFiltersForm--{{ section_id }}-{{ suffix }}"
-                {% endif %}
+              <input
+                type="radio"
+                name="sort_by"
+                id="sort-option-{{ option.value }}-{{ section_id }}"
+                {% if suffix != blank %}
+                  form="FacetFiltersForm--{{ section_id }}-{% if suffix == 'mobile' %}mobile-sorting-only{% else %}{{ suffix }}{% endif %}"
+                {% endif %}
```

This matches the user's choice (Fix suffix to match form id) and keeps `blocks/filters.liquid` untouched.

### 2. `assets/facets.js` — make `updateFacetStatus` work for radio changes

Currently (lines 681-692):

```js
updateFacetStatus(event) {
  if (!(event.target instanceof HTMLSelectElement)) return;
  // ...
  facetStatus.textContent =
    event.target.value !== details.dataset.defaultSortBy ? event.target.dataset.optionName ?? '' : '';
}
```

The status text needs to refresh whether the change came from the `<select>` (mobile) or from a radio button (desktop listbox). Replace the early-return with a normalizer that pulls `value` and `optionName` off either element type:

```js
updateFacetStatus(event) {
  const target = event.target;
  if (!(target instanceof HTMLInputElement) && !(target instanceof HTMLSelectElement)) return;

  const details = this.querySelector('details');
  if (!(details instanceof HTMLElement)) return;

  const facetStatus = details.querySelector('facet-status-component');
  if (!(facetStatus instanceof HTMLElement)) return;

  const value = target.value;
  const name = target.dataset.optionName ?? '';

  facetStatus.textContent = value !== details.dataset.defaultSortBy ? name : '';
}
```

Two small wins here:
- It now works for radio changes (the actual bug).
- It also uses the `name` from the input itself instead of `event.target.dataset.optionName` — the radio inputs at `sorting.liquid:128` already have `data-option-name="{{ option.name }}"`, so the dataset key matches what we read.

The native `facetStatus.textContent = …` keeps the existing API surface; no new methods needed.

## Why these are the right fixes

- We do not need to add a new `selectFromRow` method. The existing `updateFilterAndSorting` already does everything: form lookup, dispatching `FilterUpdateEvent`, re-rendering the section, and closing the details element. Reusing it keeps a single code path for "the user picked a sort option" regardless of input type.
- We do not need to touch `assets/component.js` — the `on:event="/method"` dispatcher is generic and works for any method name on the closest `Component` instance.
- We do not need to touch `blocks/filters.liquid` — fixing the suffix inside the snippet keeps the fix scoped to the file that owns the broken form attribute.

## Verification

1. **Desktop listbox (radios)**
   - Open any collection page (e.g. `/collections/all`) at ≥ 750 px width.
   - Click the "Sort by" pill — the listbox opens.
   - Click any option (e.g. "Price, low to high"). Expected:
     - The dropdown closes.
     - The URL updates to `?sort_by=price-ascending` (plus any active filters).
     - The product grid re-renders in the new order.
     - The summary text updates to "Price, low to high" (this exercises fix #2).
   - Before: nothing happens, an unhandled promise rejection / error is logged.

2. **Mobile select**
   - Open the same page at < 750 px width (DevTools responsive mode is fine).
   - In the select element, pick a different sort option. Expected: URL updates and grid re-renders. This was already working — confirm we haven't regressed it.

3. **Mobile sorting-only (no filtering enabled)**
   - In the customizer, disable "Enable filtering" for the filters block, leave "Enable sorting" on, save.
   - Visit a collection page on mobile.
   - Open the sort dropdown and pick a different option. Expected: URL gets the new `sort_by` and the grid re-sorts.
   - Before: the radio value was dropped on submit, so the page never re-sorted (fix #3).

4. **Regression check on filters**
   - Apply a filter (e.g. a tag) and confirm the URL still includes both the filter param and the new `sort_by`.
   - This is to confirm we didn't break the form-submission path used by `FacetsFormComponent#createURLParameters`.
