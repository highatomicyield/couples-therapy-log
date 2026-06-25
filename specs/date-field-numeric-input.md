# Spec: Date Field Input Mask and Numeric Input Modes
## couples-therapy-log — /specs/date-field-numeric-input.md

This spec replaces the native date picker input with a text field using Cleave.js
for an input mask, resolving iOS layout issues while maintaining a good mobile
entry experience. It also adds appropriate input modes to numeric fields for
better mobile keyboard behavior.

Read the current index.html in full before implementing anything in this spec.
Read all previous specs in the /specs folder for context on established patterns.

---

## 1. Cleave.js Library

### 1.1 Add CDN reference

Add the following script tag to index.html alongside the existing CDN references
(Chart.js, Google Identity Services, Google API):

```html
<script src="https://cdnjs.cloudflare.com/ajax/libs/cleave.js/1.6.0/cleave.min.js"></script>
```

Load order: add after Chart.js, before the Google library tags.

### 1.2 Offline behavior

If Cleave.js fails to load (offline), the date field falls back to a plain text
input. The rest of the app is unaffected. No special handling required — graceful
degradation is automatic.

---

## 2. Date Field Replacement

### 2.1 Problem

`input[type="date"]` has an intrinsic minimum width on iOS that cannot be
overridden by CSS, causing layout issues in the two-column form grid on
iPhone and iPad regardless of the CSS approach used.

### 2.2 Solution

Replace every `input[type="date"]` in the app with `input[type="text"]`
enhanced with a Cleave.js date mask in MM/DD/YYYY format.

### 2.3 Cleave.js initialization

After the form renders and the date input element exists in the DOM,
initialize Cleave.js on the date field:

```javascript
const cleaveDate = new Cleave('#f-date', {
  date: true,
  datePattern: ['m', 'd', 'Y'],
  delimiter: '/',
});
```

`datePattern: ['m', 'd', 'Y']` produces MM/DD/YYYY format.
`delimiter: '/'` inserts slashes automatically as the user types.

### 2.4 Pre-fill with today's date

After Cleave.js is initialized on the field, set today's date as the
initial value using Cleave's setValue method (not direct DOM manipulation,
which can conflict with the mask):

```javascript
const today = new Date();
const mm = String(today.getMonth() + 1).padStart(2, '0');
const dd = String(today.getDate()).padStart(2, '0');
const yyyy = today.getFullYear();
cleaveDate.setRawValue(`${mm}${dd}${yyyy}`); // Cleave formats this as MM/DD/YYYY
```

Note: use setRawValue() with digits only — Cleave handles inserting the slashes.

### 2.5 Select-all on focus behavior

Apply select-all on focus to the date field:

```javascript
document.getElementById('f-date').addEventListener('focus', function() {
  this.select();
});
```

Behavior: tapping the field selects the pre-filled date value. The user's
first keystroke replaces the selection and Cleave.js reformats from that
digit forward through the mask. The user types eight digits total with no
need to type slashes — the mask inserts them automatically.

Do NOT attempt to reset the field to empty mask state (`__/__/____`) on
focus — work with Cleave.js's natural behavior, not against it.

### 2.6 Date format conversion

**Input/display format:** MM/DD/YYYY (what Cleave.js produces)
**Storage format:** YYYY-MM-DD (what is written to Google Sheets)
**Log list display:** existing behavior unchanged

On save, read the raw value from Cleave and convert to storage format:

```javascript
// Get raw digits from Cleave (returns MMDDYYYY without slashes)
const raw = cleaveDate.getRawValue(); // e.g. "06252026"
const storageDate = `${raw.slice(4)}-${raw.slice(0,2)}-${raw.slice(2,4)}`; // "2026-06-25"
```

On edit pre-population, convert YYYY-MM-DD from the Sheet back to raw
digits for Cleave's setRawValue():

```javascript
function storageToRaw(yyyymmdd) {
  // Convert "2026-06-25" to "06252026" for Cleave setRawValue
  const parts = yyyymmdd.split('-');
  if (parts.length !== 3) return '';
  return `${parts[1]}${parts[2]}${parts[0]}`; // MMDDYYYY
}
cleaveDate.setRawValue(storageToRaw(entry.date));
```

### 2.7 Date usage audit

Audit every location in the codebase where date values are read, compared,
or sorted. Confirm that all such operations use the YYYY-MM-DD storage format
so that chronological sorting and date comparisons continue to work correctly.

Specific areas to check:
- Log list sort (entries sorted by date descending)
- Maintenance schedule calculations (if any date comparisons exist)
- Any date displayed in the log list item view

### 2.8 Input validation on save

On save, validate that the date field contains a complete date:

```javascript
const raw = cleaveDate.getRawValue();
if (raw.length !== 8) {
  showToast('Please enter a complete date (MM/DD/YYYY)');
  document.getElementById('f-date').focus();
  return; // abort save
}
```

Checking raw length of 8 (MMDDYYYY) is sufficient — Cleave ensures the
digits are in the correct positions.

### 2.9 Cleave instance lifecycle

The Cleave instance must be destroyed and recreated each time the form
is re-rendered (renderForm() is called), to avoid attaching multiple
instances to the same element.

Store the Cleave instance in a variable scoped outside renderForm():

```javascript
let cleaveDateInstance = null;

function initDateMask() {
  if (cleaveDateInstance) {
    cleaveDateInstance.destroy();
    cleaveDateInstance = null;
  }
  cleaveDateInstance = new Cleave('#f-date', {
    date: true,
    datePattern: ['m', 'd', 'Y'],
    delimiter: '/',
  });
  // Pre-fill today's date
  const today = new Date();
  const mm = String(today.getMonth() + 1).padStart(2, '0');
  const dd = String(today.getDate()).padStart(2, '0');
  const yyyy = today.getFullYear();
  cleaveDateInstance.setRawValue(`${mm}${dd}${yyyy}`);
  // Select-all on focus
  document.getElementById('f-date').addEventListener('focus', function() {
    this.select();
  });
}
```

Call initDateMask() after renderForm() sets the innerHTML of the form container,
once the DOM element exists.

---

## 3. Numeric Input Modes

### 3.1 Problem

On iOS, `input[type="number"]` does not consistently trigger the numeric
keypad. The `inputmode` attribute explicitly controls which keyboard
appears on mobile without changing the field's data type.

### 3.2 Fix

Add `inputmode` attributes to all numeric input fields throughout the app.

**`inputmode="decimal"`** — fields accepting decimal values:
- Gallons added (Fuel)
- Price per gallon (Fuel)
- Engine hours at fill (Fuel)
- Engine hours start / end (Trip Log)
- Fuel level start / end (Trip Log)
- Distance nm (Trip Log)
- House bank voltage (Daily Checklist)
- Cost fields (Maintenance, Pumpout, Parts rows)
- Any other field accepting dollars or decimal measurements

**`inputmode="numeric"`** — fields accepting whole numbers only:
- House bank SOC % (Daily Checklist)
- Freshwater level % (Daily Checklist)
- Fuel level % (Daily Checklist)

### 3.3 Scope

Apply to every numeric input rendered by renderForm() and by the Parts
row generation logic. Do not apply to text fields, select elements,
textareas, or the date field (which uses Cleave.js keyboard handling).

---

## 4. Implementation Sequence

1. Add Cleave.js CDN script tag (Section 1.1)
2. Create cleaveDateInstance variable and initDateMask() function (Section 2.9)
3. Replace all `input[type="date"]` with `input[type="text"]` in renderForm()
4. Call initDateMask() after each renderForm() call
5. Update save logic to use getRawValue() and convert to YYYY-MM-DD (Section 2.6)
6. Update edit pre-population to use setRawValue() with storageToRaw() (Section 2.6)
7. Audit all date comparisons and sort operations (Section 2.7)
8. Add save validation for complete date (Section 2.8)
9. Add inputmode attributes to all numeric fields (Section 3)

---

## 5. Out of Scope for This Prompt

- Any changes to form fields beyond date and inputmode
- Summary tab enhancements
- Maintenance tab enhancements
- Any changes to Google Sheets structure
