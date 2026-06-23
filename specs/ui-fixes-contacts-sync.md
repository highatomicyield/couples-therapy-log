# Spec: UI Fixes, Contacts Tab, and Sync Button
## couples-therapy-log — /specs/ui-fixes-contacts-sync.md

This spec covers a date field width fix on mobile, a new Contacts tab in Google Sheets
for the Performed By field in Maintenance entries, and a sync button in the header.

Read the current index.html in full before implementing anything in this spec.
Read all previous specs in the /specs folder for context on the data model
and patterns already established.

---

## 1. Date Field Width Fix

### 1.1 Problem

The date input field renders wider than other form fields on mobile (iPhone and iPad),
causing layout inconsistency in the entry forms.

### 1.2 Fix

Constrain the date input field width to 100% of its parent grid column,
consistent with all other input fields. Override any browser default intrinsic sizing.

Add to the CSS:

```css
input[type="date"] {
  width: 100%;
  box-sizing: border-box;
}
```

Verify this renders consistently across the form grid on both mobile and desktop
viewport widths.

---

## 2. Contacts Tab

### 2.1 New Sheet tab: Contacts

Create a new Sheet tab named **Contacts** with a single column header: **name**

Populate with the following initial values, one per row, in this order:

1. Gig Harbor Marina & Boatyard
2. Owner
3. ReVolt Marine Systems

Sorted alphabetically. "Other" is NOT a row in this tab — append it
programmatically in the dropdown, same pattern as Vendors tab.

### 2.2 App initializes Contacts tab on first run

Following the same ensureSheetHeaders pattern used for Log, Parts, Vendors,
and Preferences tabs:
- Check whether the Contacts tab exists and has the correct header row
- If not, create it with the name header and populate the initial contact list
- If it exists, leave it unchanged

### 2.3 Reading Contacts at sign-in

Read the Contacts tab at sign-in alongside Vendors and Preferences.
Store as an array in memory, same pattern as vendors list.
Use to populate the Performed By dropdown in Maintenance entry forms.

### 2.4 Performed By field — change from static dropdown to Contacts-driven dropdown

Replace the current hardcoded Performed By dropdown in the Maintenance entry form
with a dynamic dropdown populated from the Contacts tab, same implementation
pattern as the Source dropdown in the Parts UI:

- Dropdown options: all entries from Contacts tab, in the order they appear in the tab
- "Other" appended at the bottom programmatically — not stored in the Contacts tab
- Selecting "Other" reveals a free-text input field immediately below the dropdown
- The free-text value is stored in the Log tab `who` column, not the word "Other"
- If "Other" is selected and free-text is left blank, store "Other"

### 2.5 SHEET_COLUMNS

No change to SHEET_COLUMNS required. The `who` field already exists in the Log tab
and stores the performed-by value as a string. This change only affects how that
value is collected in the UI.

### 2.6 Edit mode pre-population

When opening a Maintenance entry for editing:
- If the stored `who` value matches a name in the Contacts tab, select it in the dropdown
- If not, select "Other" and populate the free-text field with the stored value
- Same pattern as Source field in Parts UI (established in variable-length-parts-crew.md)

---

## 3. Sync Button

### 3.1 Placement

Add a small sync button to the site header, positioned immediately to the left
of the Sign Out button. Same vertical alignment as Sign Out.

### 3.2 Appearance

- Icon: circular arrow (↻) — use an inline SVG consistent with other icons in the app
- Size: slightly wider than the arrow icon diameter, same height as Sign Out button
- Style: match the Sign Out button appearance (same background, border, text color)
- No label text — icon only
- Tooltip on hover (desktop): "Sync with Google Sheets"
- Only visible when signed in — hidden on auth screen, same as Sign Out button

### 3.3 Sync behavior

On click:
1. Trigger a full re-read from all Sheet tabs: Log, Parts, Vendors, Contacts, Preferences
2. Update all in-memory data (entries array, vendors list, contacts list, prefs object)
3. Update localStorage cache via persistToCache()
4. Re-render the current tab view with fresh data
5. Show toast: "Synced" — same toast as shown after initial sign-in data load

This is identical to the data loading sequence that runs after successful authentication,
extracted into a reusable function that both the sign-in flow and the sync button call.

### 3.4 Error handling

If any Sheet read fails during sync:
- Show toast: "Sync failed — showing cached data"
- Do not update in-memory data or cache with partial results
- Leave current view unchanged

---

## 4. Implementation Sequence

1. Date field CSS fix (Section 1)
2. Contacts tab ensureSheetHeaders extension (Section 2.2)
3. Contacts tab read at sign-in (Section 2.3)
4. Performed By field — replace static dropdown with Contacts-driven dropdown (Section 2.4)
5. Extract sign-in data loading into a reusable sync function (Section 3.3 note)
6. Sync button — header placement and appearance (Section 3.1, 3.2)
7. Sync button — behavior wired to reusable sync function (Section 3.3)

---

## 5. Out of Scope for This Prompt

- Summary tab enhancements
- Maintenance tab enhancements
- Any new entry types or form fields
- Any changes to Google Sheets column structure beyond the Contacts tab
- Pull-to-refresh gesture (explicitly out of scope — sync button is the chosen solution)
