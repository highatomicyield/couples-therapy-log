# Spec: Variable-Length Parts List and Crew List
## couples-therapy-log — /specs/variable-length-parts-crew.md

This spec covers two related features: a variable-length parts list on Maintenance entries
and a variable-length crew list on Trip Log entries. Both require changes to the Google
Sheets data model and the entry form UI.

---

## 1. Google Sheets — New and Modified Tabs

### 1.1 New tab: Parts

Create a new Sheet tab named **Parts** with the following column headers in this exact order:

| Column | Description |
|--------|-------------|
| entry_id | ID of the parent Maintenance log entry (matches `id` field in Log tab) |
| description | Part or material description |
| part_number | Manufacturer or supplier part number (optional) |
| cost | Cost of this line item in dollars (numeric) |
| source | Supplier name (from Vendors tab, or free-text if Other) |

- One row per part per maintenance entry
- Multiple rows may share the same entry_id
- Do not add a quantity field

### 1.2 New tab: Vendors

Create a new Sheet tab named **Vendors** with a single column header: **name**

Populate with the following initial values, one per row, in this order:

1. Boat Outfitters
2. Fisheries Supply
3. Gig Harbor Marina & Boatyard
4. Great Lakes Skipper
5. Home Depot
6. Longship Marine
7. Pacific Power Group
8. Panther RV Products
9. ReVolt Marine Systems
10. West Marine

- Sorted alphabetically for easy scanning
- "Other" is NOT a row in this tab — it is appended programmatically by the app
- To add a new vendor in the future, add a row to this tab — no code changes required
- The app reads this tab at sign-in to populate the source dropdown in the Parts UI

### 1.3 Modified tab: Log

Add the following columns to the END of the existing SHEET_COLUMNS array in index.html.
Do not change the position of any existing columns:

- crew1
- crew2
- crew3
- crew4
- crew5
- crew6
- crew7
- crew8
- crew9
- crew10

These columns store crew member names (including pets) for Trip Log entries.
Empty columns for unused crew slots are written as empty strings.

---

## 2. Maintenance Form — Parts UI

### 2.1 Collapsed by default

The parts section appears below the main Maintenance form fields, collapsed by default.
A single button labeled **+ Add parts** expands it.

Once expanded, the section displays a list of part rows and cannot be re-collapsed
(to prevent accidental data loss mid-entry).

### 2.2 Parts row UI

Each part row contains the following fields in a single horizontal line or compact grid:

- **Description** — text input, required
- **Part number** — text input, optional
- **Cost** — numeric input, optional, dollar amount
- **Source** — dropdown populated from Vendors tab, with "Other" appended at the bottom

When "Other" is selected in the Source dropdown:
- A free-text input field appears immediately below (or inline) the dropdown
- The free-text value is stored in the Parts tab `source` column, not the word "Other"
- If the free-text field is left blank and Other is selected, store "Other"

Each row has a **remove** button (× or trash icon) to delete that row from the form.
Removing a row does not affect already-saved entries.

A button labeled **+ Add another part** appears below the last row to append a new empty row.

### 2.3 Saving parts

When the user saves a Maintenance entry:
1. The main entry is written to the Log tab as usual
2. Each parts row is written as a separate row to the Parts tab, with entry_id set to
   the id of the just-saved Log entry
3. Empty parts rows (no description) are skipped and not written to the Sheet

### 2.4 Displaying parts in the log list

When a Maintenance entry is rendered in the Log tab view, if it has associated parts rows
in the Parts tab, display them as a simple line-item list below the main entry detail line.
Format: "Description — Source — $Cost"

Parts are loaded from the Parts tab by matching entry_id to the Log entry's id field.
Load parts for visible log entries after the main log list renders (background fetch).

### 2.5 System dropdown addition

Add **Various** as an option in the Maintenance form System dropdown.
Position it at the end of the existing list.

---

## 3. Trip Log Form — Crew UI

### 3.1 Behavior

The crew section appears below the main Trip Log form fields.
It starts with a single empty name field labeled **Crew member / pet**.
A button labeled **+ Add crew** appends additional name fields, up to a maximum of 10.

Each name field has a **remove** button to delete that row.
The first row cannot be removed if it is the only row (to maintain at least one field visible).
If all crew fields are left empty, no crew data is written — this is valid for a solo trip.

### 3.2 Saving crew

Crew names are written to the crew1 through crew10 columns in the Log tab.
First name field → crew1, second → crew2, and so on.
Unused slots are written as empty strings.

### 3.3 Displaying crew in the log list

If crew fields are present on a Trip Log entry, display names as a comma-separated
list in the log entry detail line. Example: "John, Julie, Otis"

---

## 4. Future Edit/Delete Compatibility — Important Notes for Implementation

This section documents constraints that must be respected when edit and delete
functionality is implemented in a future prompt. Build the Parts and Crew structures
with these in mind now:

- **Deleting a Maintenance entry** must also delete all rows in the Parts tab
  where entry_id matches the deleted entry's id
- **Editing a Maintenance entry's parts** requires identifying existing Parts tab rows
  by entry_id, then reconciling additions, changes, and removals against those rows
- **The entry id field** is the sole join key between Log and Parts tabs —
  it must be written reliably and must be unique. Confirm this is the case in the
  current implementation before building the Parts write logic
- **Crew edit** is simpler — crew1 through crew10 are columns in the Log tab row
  itself, so editing a trip entry updates those columns in place like any other field

---

## 5. Implementation Sequence

Implement in this order to allow testing at each stage:

1. Create Vendors tab and populate initial list
2. Create Parts tab with correct headers
3. Add crew1–crew10 to SHEET_COLUMNS and Log tab header row
4. Implement Vendors tab read at sign-in
5. Implement Parts UI on Maintenance form (collapsed, add/remove rows, Other handling)
6. Implement Parts write on Maintenance entry save
7. Implement Parts display in log list view
8. Implement Crew UI on Trip Log form
9. Implement Crew write on Trip Log entry save
10. Implement Crew display in log list view
11. Add Various to System dropdown

---

## 6. Out of Scope for This Prompt

- Edit and delete functionality (future prompt)
- Summary tab changes (future prompt)
- Any changes to Fuel, Pumpout, Systems Check, or Note entry types
