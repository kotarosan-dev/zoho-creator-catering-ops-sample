# Zoho Creator — Catering Operations Sample

A self-built sample application in Zoho Creator that turns one confirmed catering
booking (guest count + selected menu) into the numbers the operation runs on:
ingredient quantities, shopping lists, a prep sheet, quote totals, deposit and balance.

> **What this is / is not**
> This is a personal sample I designed and built on my own Zoho account with dummy
> data, based on the requirements of a real catering-operations job posting.
> It is **not** a paid client project. Feel free to reuse the code and the design.

## Demo video

[Watch the walkthrough (1:44, YouTube — unlisted)](https://youtu.be/KhbDTMjKopY)
— a new booking (80 guests) is entered on the live app, two menu lines are added,
and the quote, deposit, prep quantities and shopping list are calculated
automatically. Every number shown was checked against hand calculations.

## What works today (Phase 1, verified end-to-end)

- 10 forms: `Settings`, `Menu_Items`, `Ingredients`, `Customers`, `Recipes`,
  `Bookings`, `Booking_Lines`, `Booking_Calculations`, `Prep_Sheet_Items`,
  `Shopping_List_Items`
- Deluge function `calculateBooking(bookingId)`:
  - per-item quantities from guest count × per-guest servings, with a buffer rate
  - ingredient requirements aggregated from per-guest recipes
  - shopping quantities rounded **up to whole purchase units**
    (e.g. 220 tortillas needed → buy 250, because they come in packs of 50)
  - shopping list rows tagged Perishable / Non-Perishable
  - money: subtotal → service fee → tax → total → deposit / balance, written back
    onto the booking
  - safe re-calculation: previous results are cleared in bulk before re-writing
- A form workflow re-runs the calculation every time a booking line is added.

Verified with a 50-guest test booking (2 menu items):
subtotal $1,300.00, service fee $130.00, tax $114.40, **total $1,544.40**,
deposit $772.20 — matching hand-computed values exactly, including purchase-unit
rounding (chicken 8.25 kg → 10 kg, limes 27.5 → 40).

## Phase 2 additions (built and verified)

- 13 forms: the Phase 1 set plus `Packing_Rules`, `Packing_Items`, `Expenses`
- Deluge function `buildPackingList(bookingId)`:
  - generates the packing checklist from a rules table — four calculation
    types: Fixed, Per Guest, Per Guest Block (e.g. one warming box per
    25 guests), Per Menu Unit (scaled by the prep quantity)
  - rules are evaluated in order and snapshot into the generated rows, so
    renaming or deleting a rule later never rewrites an issued list
  - regeneration is blocked once any item is checked off, so field progress
    is never wiped by a recalculation
- Deluge function `profitSummary(bookingId)`:
  - per-booking profit estimate: revenue excluding tax − ingredient purchase
    cost (purchase quantity × unit cost) − recorded expenses
  - refreshes automatically when an expense is added or edited
  - a missing ingredient cost fails the calculation instead of silently
    counting as zero, and a failed run never overwrites the last good numbers

Verified on the same 50-guest test booking: revenue (excl. tax) $1,430.00,
ingredient purchase cost $142.00, expenses $0 — **gross profit $1,288.00,
margin 90.1%**. All 13 Phase 2 test cases pass (17 checks: TC-K1–K9, TC-P1–P8).

## Data model

See [docs/data-model.md](docs/data-model.md). The core idea: a **recipe table**
(menu item × ingredient × quantity per guest) is the single source from which
prep quantities, shopping quantities and costs are all derived.

## Two lessons baked into this sample

1. **SubForms are not always available.** On the Creator plan used here, a form
   containing a SubForm could not be submitted at all ("Your current plan allows
   only demo of the SubForm"). Booking lines therefore live in a standalone
   `Booking_Lines` form with a Lookup to `Bookings` — a pattern that also scales
   better for reporting.
2. **Never delete rows inside a `for each` over the same collection.** It fails at
   runtime ("invalid expressions delete statement is terminated") once more than
   one row matches. The function uses bulk deletes by criteria instead.

## Designed, not yet built

- Invoices (deposit / final)
- Integrations with Zoho CRM / Sign / Books and Square deposits
  (designed in the data model, not implemented in this sample)

3. **Do not name a loop variable after the collection you insert into.**
   Reusing the same name fails only when rows already exist — the first
   generation passes, regeneration throws. Test the second run, not the first.
4. **A Creator Checkbox field is not a boolean.** It returns the list of
   selected options, so criteria and conditions have to be written against
   a list.

## Files

- `deluge/calculateBooking.ds` — the calculation engine (installed and verified)
- `deluge/buildPackingList.ds` — packing checklist generation (Phase 2, installed and verified)
- `deluge/profitSummary.ds` — per-booking profit estimate (Phase 2, installed and verified)
- `docs/data-model.md` — forms, fields and calculation rules

Built by Kotaro Izumida — Zoho CRM / Creator consultant
(Upwork: https://www.upwork.com/freelancers/~01a5ef53de58789fdc)
