# Spec: Edit and Delete Log Entries
## couples-therapy-log — /specs/edit-delete-entries.md

This spec covers edit and delete functionality for all log entry types.
It builds on the data model established in variable-length-parts-crew.md.
Read the current index.html in full before implementing anything in this spec.

---

## 1. Overview of Behavior

- Tapping or clicking any entry in the log list opens it for editing
- The existing New Entry form reopens pre-populated with the entry's current values
- The Save Entry button overwrites the existing Sheet row rather than appending a new one
- A red Delete Entry button at the bottom of the form triggers a confirmation prompt
- On confirmation, the entry and any associated data are permanently removed from the Sheet

---

## 2. Entry Point — Log List

### 2.1 Tap/click to edit

Every entry in the log list is tappable/clickable.
On tap/click:
1. Switch to the New Entry tab
2. Select the correct entry type button to match the entry being edited
3. Populate all form fields with the entry's current values
4. Set a flag (e.g. `editingEntryId`) to indicate this is an edit, not a new entry
5. Change the Save Entry button label to **Update Entry** to signal edit mode
6. Show the Delete Entry button (hidden during new entry creation)

### 2.2 Canceling an edit

Add a **Cancel** button that appears alongside Update Entry and Delete Entry during edit mode.
Cancel returns the form to a blank new entry state and clears the editingEntryId flag.
No changes are written to the Sheet on cancel.

---

## 3. Save / Update Behavior

### 3.1 New entry (existing behavior — do not change)

When editingEntryId is null:
- Append a new row to the Log tab
- Append parts rows to the Parts tab if applicable
- Behavior unchanged from current implementation

### 3.2 Edit / update (new behavior)

When editingEntryId is set:

**For all entry types:**
- Find the existing row in the Log tab where the id column matches editingEntryId
- Overwrite that row in place with the updated field values
- Do not change the entry's id value — it must remain stable as a join key

**For Maintenance entries additionally:**
- Delete all existing rows in the Parts tab where entry_id matches editingEntryId
- Rewrite Parts tab rows from the current form state (delete-and-rewrite pattern)
- Empty parts rows (no description) are skipped

**For Trip Log entries additionally:**
- crew1 through crew10 columns are updated in place as part of the Log tab row overwrite
- No secondary tab cleanup needed for crew

After a successful update:
- Clear editingEntryId flag
- Return form to blank new entry state
- Refresh log list from Sheet
- Show toast: "Entry updated"

---

## 4. Delete Behavior

### 4.1 Delete Entry button

Displayed at the bottom of the form during edit mode only.
Styled in red to signal a destructive action.
Never visible during new entry creation.

### 4.2 Confirmation prompt

On click, display a confirmation dialog before taking any action.
Suggested text:
"Delete this entry? This cannot be undone."
Two buttons: **Delete** (red) and **Cancel**.
On Cancel: dismiss dialog, return to edit form, no action taken.
On Delete: proceed with deletion as described below.

### 4.3 Deletion logic

**For all entry types:**
- Find and delete the row in the Log tab where id matches editingEntryId
- Deleting a Sheet row means clearing its contents and removing the row entirely,
  not just blanking the cells — use the spreadsheets.batchUpdate method with
  a DeleteDimensionRequest to remove the row

**For Maintenance entries additionally:**
- Find and delete all rows in the Parts tab where entry_id matches editingEntryId
- Same deletion approach — remove rows entirely, not just blank them

**For all other entry types:**
- Log tab row deletion only, no secondary tab cleanup needed

After successful deletion:
- Clear editingEntryId flag
- Remove entry from local cache
- Return form to blank new entry state
- Refresh log list
- Show toast: "Entry deleted"

---

## 5. Pre-population of Form Fields by Entry Type

When an entry is opened for editing, populate fields as follows:

**All entry types:**
- Date, location, notes — populate from entry values

**Fuel:**
- Gallons, price per gallon, engine hours — populate from entry values
- Recalculate and display total cost derived field on load

**Pumpout:**
- Holding tank status, cost — populate from entry values

**Maintenance:**
- Engine hours, system, task, who, cost — populate from entry values
- Parts section: expand automatically (do not leave collapsed) if the entry
  has associated rows in the Parts tab
- Load parts rows from Parts tab by entry_id and populate as individual rows
  in the parts UI, each with description, part number, cost, and source pre-filled
- If source value matches a vendor in the Vendors tab, select it in the dropdown;
  if not, select Other and populate the free-text field with the stored value

**Trip Log:**
- From, to, engine hours start/end, fuel level start/end, sea state — populate from entry values
- Recalculate and display derived fields (fuel consumed, hours underway, burn rate) on load
- Crew: populate crew name fields from crew1 through crew10 columns,
  skipping empty values, showing one field per non-empty crew member

**Systems check:**
- All dropdowns and numeric fields — populate from entry values
- Electronics status dropdowns — populate from entry values

**Note:**
- Notes textarea — populate from entry value

---

## 6. Finding Rows in the Sheet

The Sheet API does not provide a direct "find row by value" method.
The implementation must:
1. Read all rows from the relevant tab
2. Find the row index where the id column matches the target value
3. Use that row index for update or delete operations

Row indices in the Sheets API are zero-based for batchUpdate operations
but the data range starts at row 2 (row 1 is headers).
Account for this offset carefully to avoid updating or deleting the wrong row.

This lookup pattern is needed in three places:
- Finding a Log tab row to update
- Finding a Log tab row to delete
- Finding Parts tab rows to delete (may return multiple rows)

Consider implementing a reusable helper function for row lookup to avoid
duplicating this logic.

---

## 7. Local Cache Consistency

The localStorage cache must stay consistent with Sheet operations:

**On update:** replace the entry in the entries array where id matches,
then call persistToCache()

**On delete:** remove the entry from the entries array where id matches,
then call persistToCache()

**Parts are not cached locally** — they are fetched from the Sheet on demand
when a log entry is expanded or opened for editing.

---

## 8. Error Handling

All Sheet API calls must be wrapped in try/catch.

**On update failure:**
- Show toast: "Update failed — changes not saved"
- Do not clear editingEntryId — leave form open so user can retry
- Do not update local cache

**On delete failure:**
- Show toast: "Delete failed — entry not removed"
- Do not clear editingEntryId — leave form open so user can retry
- Do not update local cache

---

## 9. Out of Scope for This Prompt

- Summary tab enhancements
- Vendor management UI (adding vendors from within the app)
- Any changes to form fields or entry types
- Offline/cache-only edit or delete (all edits and deletes require an active
  Sheet connection — if accessToken is null, show toast: "Sign in required to edit or delete entries")
