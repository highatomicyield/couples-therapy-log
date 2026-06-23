# Spec: UI Tweaks, Preferences Tab, and Edit Mode Fixes
## couples-therapy-log — /specs/ui-tweaks-preferences-edit-fixes.md

This spec covers edit mode behavioral fixes, a new Preferences tab in Google Sheets
for personal defaults, select-all on focus for pre-filled fields, placeholder text
standardization, sticky departure default for Trip Log, and parts cost auto-sum
in Maintenance entries.

Read the current index.html in full before implementing anything in this spec.
Read specs/variable-length-parts-crew.md and specs/edit-delete-entries.md for
context on the data model and edit/delete implementation this spec builds on.

---

## 1. Edit Mode Fixes

### 1.1 Entry type locked during edit

When `editingEntryId` is set (edit mode is active):
- All six entry type buttons (Fuel, Pumpout, Maintenance, Trip Log, Systems Check, Note)
  must be disabled
- Disabled state: greyed out / reduced opacity (0.4), cursor set to not-allowed
- Clicking a disabled type button does nothing — no type switch, no form change
- Type buttons return to normal active state when edit mode is cancelled or completed

### 1.2 Edit mode cancelled on tab navigation

When the user navigates away from the New Entry tab while `editingEntryId` is set:
- Automatically cancel edit mode
- Clear `editingEntryId`
- Reset the form to a blank new entry state for the current default entry type (Fuel)
- Do not show a confirmation prompt — the explicit Cancel button already handles
  intentional cancellation
- This applies to navigation to any tab: Log, Maintenance, Summary

### 1.3 Auto-return to Log tab after update or delete

After a successful Update Entry operation:
- Clear edit mode
- Automatically switch to the Log tab
- Refresh the log list so the updated entry is visible
- Show toast: "Entry updated"

After a successful Delete Entry operation:
- Clear edit mode
- Automatically switch to the Log tab
- Refresh the log list with the entry removed
- Show toast: "Entry deleted"

---

## 2. Preferences Tab

### 2.1 New Sheet tab: Preferences

Create a new Sheet tab named **Preferences** with two columns: **key** and **value**

Populate with the following initial rows:

| key | value |
|-----|-------|
| default_location | Port of Brownsville |
| default_crew1 | (leave blank — to be filled in manually by user after deployment) |

**Important:** do not hardcode any personal names or identifying information in
index.html. The Preferences tab is the only place personal defaults are stored,
and it is protected behind Google authentication.

### 2.2 Reading preferences at sign-in

After successful authentication, read the Preferences tab alongside the Vendors tab.
Store preferences in a `prefs` object keyed by the key column value.

Example: `prefs['default_location']` returns `'Port of Brownsville'`

If the Preferences tab does not exist or a key is missing, fail gracefully —
use empty string as fallback, do not throw an error.

### 2.3 App should initialize Preferences tab on first run

Following the same ensureSheetHeaders pattern used for Log, Parts, and Vendors tabs:
- Check whether the Preferences tab exists and has the correct header row
- If not, create it with key/value headers and the default_location row
- Leave default_crew1 row with a blank value — the user fills it in manually
  in Google Sheets after deployment

### 2.4 Applying preferences to forms

**default_location:**
- Applied to the Location field on all entry types except Trip Log
- Pre-filled as black text (not placeholder), value from prefs['default_location']
- Select-all on focus behavior applied (see Section 4)
- If preference value is empty string, field renders empty with placeholder text

**default_crew1:**
- Applied to the first Crew field (Crew 1) on Trip Log entries only
- Pre-filled as black text, value from prefs['default_crew1']
- Select-all on focus behavior applied (see Section 4)
- If preference value is empty string, field renders empty with placeholder text

---

## 3. Sticky Departure Default for Trip Log

### 3.1 Behavior

The Departure Point field on Trip Log entries pre-fills with the Destination value
from the most recently saved Trip Log entry.

This is stored in localStorage (device-specific) under the key `ct_last_destination`.
Cross-device sync is not required for this feature — it is a convenience default,
not a critical data field.

### 3.2 Implementation

On save of any Trip Log entry:
- Write the Destination field value to localStorage as `ct_last_destination`
- Overwrite any previous value

On rendering the Trip Log form:
- Read `ct_last_destination` from localStorage
- If present, pre-fill the Departure Point field with that value
- Apply select-all on focus behavior (see Section 4)
- If not present (first Trip Log entry ever), render empty with placeholder text

### 3.3 Known limitation

This default is device-specific. If the user logs a trip on mobile, the desktop
browser will not reflect that destination as the next departure point until a trip
is logged on desktop as well. This is acceptable — document as a known limitation
in a code comment near the implementation.

---

## 4. Select-All on Focus for Pre-Filled Fields

Apply select-all on focus behavior to the following fields:

- **Location** field on all entry types (pre-filled from Preferences)
- **Crew 1** field on Trip Log entries (pre-filled from Preferences)
- **Departure Point** field on Trip Log entries (pre-filled from sticky default)
- **Cost** field on Pumpout entries (pre-filled as 0)

Implementation: add a focus event listener to each affected input field after
the form renders. On focus, call `this.select()` to select all text in the field.
The user's first keystroke replaces the entire pre-filled value.

```javascript
// Example pattern — apply to each pre-filled field after form renders
document.getElementById('f-fieldname').addEventListener('focus', function() {
  this.select();
});
```

---

## 5. Placeholder Text Standardization

Apply globally across all entry type forms:

- All placeholder text must be prefixed with "e.g." where not already present
- This applies to every text input and textarea placeholder attribute throughout
  the app
- Does not apply to numeric fields (placeholder numbers like "0.0" or "100"
  do not need the "e.g." prefix)
- Does not apply to dropdown fields (no placeholder text)
- Does not apply to pre-filled fields covered in Section 4 (those have real values,
  not placeholder text)

Review every form field in renderForm() and update placeholder attributes accordingly.

---

## 6. Trip Log — Departure Point Placeholder Text

The Departure Point field placeholder text should read:
**"e.g. Port of Brownsville"**

This is placeholder text (grey, not pre-filled) for cases where the sticky default
from Section 3 is not yet set (first-ever Trip Log entry on this device).
Once the sticky default is set, the field will show the pre-filled last destination
instead of this placeholder.

---

## 7. Parts Cost Auto-Sum in Maintenance Entries

### 7.1 Behavior

When one or more parts rows are present in the Maintenance entry form:
- The Cost field becomes read-only
- Its value is automatically calculated as the sum of all parts row Cost values
- Recalculates whenever any parts row Cost value changes (on blur of any parts cost field)
- Displays as a formatted dollar amount

When no parts rows are present (parts section collapsed or all rows removed):
- The Cost field returns to manually editable
- Any previously auto-calculated value is cleared
- User enters total cost manually as before

### 7.2 Visual treatment

When auto-summing is active:
- Cost field background changes subtly to indicate read-only state
  (e.g. light grey background, same pattern used for calculated fields
  on Fuel and Trip Log forms)
- A small label or note appears near the field: "Summed from parts"
- When parts section is collapsed or empty, this label disappears and
  field returns to normal editable appearance

### 7.3 Labor and taxes as parts line items

No special handling needed. The user may add "Labor" or "Taxes" as part
descriptions with associated costs. These sum into the total Cost field
like any other parts line item. No code change required — document this
as the intended pattern in a code comment near the parts UI implementation.

### 7.4 Storage

The summed Cost value is what gets stored in the Log tab cost column.
Individual parts costs are stored in the Parts tab as before.
No change to SHEET_COLUMNS or storage logic beyond what already exists.

---

## 8. Implementation Sequence

Implement in this order:

1. Edit mode fixes (Section 1) — type locking, tab navigation cancel, auto-return to Log
2. Preferences tab creation and ensureSheetHeaders extension (Section 2.3)
3. Preferences read at sign-in (Section 2.2)
4. Apply default_location pre-fill to all applicable forms (Section 2.4)
5. Apply default_crew1 pre-fill to Trip Log crew field (Section 2.4)
6. Sticky departure default — localStorage read/write (Section 3)
7. Select-all on focus for all affected fields (Section 4)
8. Placeholder text standardization — "e.g." prefix (Section 5)
9. Parts cost auto-sum in Maintenance form (Section 7)

---

## 9. Out of Scope for This Prompt

- Summary tab enhancements
- Maintenance tab enhancements
- Any new entry types or form fields
- Vendor management UI
- Any changes to Google Sheets column structure beyond the Preferences tab
