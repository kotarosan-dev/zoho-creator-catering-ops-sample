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

Phase 2 (designed, not built): Packing_Items, Invoices (Deposit / Final), Expenses.

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
