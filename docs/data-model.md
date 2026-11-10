# Data model — Catering Operations Sample (Zoho Creator)

## Forms

| Form | Role | Key fields |
|---|---|---|
| Settings | Rates used by the calculation | Tax_Rate_Percent, Deposit_Rate_Percent, Buffer_Rate_Percent, Service_Fee_Rate_Percent |
| Menu_Items | Menu master | Item_Name, Category, Price_Per_Serving, Cost_Per_Serving, Unit |
| Ingredients | Ingredient master | Ingredient_Name, Category (Perishable / Non-Perishable), Unit, Purchase_Unit_Size, Cost_Per_Unit |
| Recipes | **The core table**: menu item × ingredient | Menu_Item (Lookup), Ingredient (Lookup), Quantity_Per_Serving, Unit |
| Customers | Customer master | Customer_Name, Company, Email, Phone |
| Bookings | One order | Customer (Lookup), Event_Date, Service_Type (Drop-off / Full Service), Guest_Count, Status (Quoted / Confirmed / Completed / Cancelled), Subtotal, Service_Fee, Tax, Total_Amount, Deposit_Amount, Balance_Amount |
| Booking_Lines | Order lines (standalone form, Lookup pattern — see README lesson 1) | Booking (Lookup), Menu_Item (Lookup), Servings_Per_Guest, Notes |
| Booking_Calculations | Calculation detail per menu item | Booking, Menu_Item, Guest_Count, Quantity_Required, Unit_Price, Line_Amount |
| Prep_Sheet_Items | What the kitchen preps | Booking, Menu_Item, Quantity_To_Prep, Unit, Notes |
| Shopping_List_Items | What to buy, split by perishability | Booking, Ingredient, Category, Required_Quantity, Purchase_Quantity, Unit |

| Packing_Rules | Packing checklist rules master | Rule_Name, Category (Equipment / Food Container / Consumable / Serving), Applies_To (All Bookings / Service Type / Menu Item), Service_Type, Menu_Item (Lookup), Calculation_Type (Fixed / Per Guest / Per Guest Block / Per Menu Unit), Base_Quantity, Block_Size, Unit, Sort_Order, Active (Checkbox), Notes |
| Packing_Items | The generated packing checklist per booking | Booking (Lookup), Item_Name (snapshot of Rule_Name), Category, Quantity_Required, Unit, Checked (Checkbox), Source_Rule (Lookup → Packing_Rules), Rule_Notes, Sort_Order, Notes |
| Expenses | Costs recorded against a booking | Booking (Lookup), Expense_Date, Category (Labor / Transport / Rentals / Extra Ingredients / Other), Description, Amount, Notes |

Phase 2 also adds 10 fields to Bookings: Revenue_Excl_Tax, Ingredient_Purchase_Cost,
Other_Expenses_Total, Gross_Profit (currency), Profit_Margin_Percent (decimal),
Profit_Status, Profit_Calculated_At, Packing_Generated_At, Calc_Status, Calc_Completed_At.

Still designed, not built: Invoices (Deposit / Final).

## Calculation rules (calculateBooking)

1. For each booking line: `quantity = guests × servings_per_guest × (1 + buffer_rate)`.
2. Line amount: `price_per_serving × guests × servings_per_guest` (buffer does not change the price).
3. Ingredients are aggregated across all lines from Recipes, with the same buffer applied.
4. Purchase quantity = required quantity rounded **up** to a multiple of the
   ingredient's `Purchase_Unit_Size` (falls back to whole units when no size is set).
5. Money: `service_fee = subtotal × service_rate`; `tax = (subtotal + service_fee) × tax_rate`;
   `total = subtotal + service_fee + tax`; `deposit = total × deposit_rate`; `balance = total − deposit`.
6. Re-calculation first bulk-deletes the booking's previous rows in the three
   result forms, then writes fresh rows — no duplicates, no stale numbers.

## Trigger

Form workflow on Booking_Lines ("on successful submission") calls
`calculateBooking(input.Booking)`, so totals are always in sync with the lines.

## Calculation rules (buildPackingList)

Preconditions: the booking's Status is Quoted or Confirmed and its
Calc_Status from calculateBooking is Complete. All validation happens
before anything is deleted.

1. Active Packing_Rules are evaluated in Sort_Order. A rule applies when
   its Applies_To matches: All Bookings always; Service Type when it
   equals the booking's Service_Type; Menu Item when that item is on the
   booking. Matching rules are additive — each produces its own row,
   traceable through Source_Rule.
2. Quantity by Calculation_Type, always rounded up to a whole number:
   Fixed = Base_Quantity; Per Guest = Base_Quantity × Guest_Count;
   Per Guest Block = ceil(Guest_Count ÷ Block_Size) × Base_Quantity;
   Per Menu Unit = Base_Quantity × the total Quantity_To_Prep of that
   menu item on the prep sheet (Per Menu Unit is only valid together
   with Applies_To = Menu Item).
3. If any Packing_Items row for the booking is already checked,
   generation refuses and changes nothing — checked progress is never
   wiped. Otherwise all rows are built in memory first, then the old
   (unchecked) rows are bulk-deleted and the new rows inserted;
   hand-written Notes carry over to a new row when Item_Name and
   Source_Rule match.

## Calculation rules (profitSummary)

Precondition: the booking's Calc_Status from calculateBooking is Complete.

1. Revenue_Excl_Tax = Subtotal + Service_Fee.
2. Ingredient_Purchase_Cost = Σ over Shopping_List_Items of
   Purchase_Quantity × the ingredient's Cost_Per_Unit. A shopping row whose
   ingredient has no cost recorded fails the whole calculation — it is
   never treated as zero.
3. Other_Expenses_Total = Σ Expenses.Amount for the booking.
4. Gross_Profit = Revenue_Excl_Tax − Ingredient_Purchase_Cost −
   Other_Expenses_Total; Profit_Margin_Percent = Gross_Profit ÷
   Revenue_Excl_Tax × 100, rounded to one decimal.
5. A failed run sets Profit_Status to the failure reason and leaves the
   last good values untouched. A form workflow on Expenses re-runs
   profitSummary whenever an expense is added or edited (deleting an
   expense needs a manual re-run; Profit_Calculated_At shows the freshness).
