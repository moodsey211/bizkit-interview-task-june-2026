# Notes

## Task 1: The double-booking bug

The overlap check was one-sided. `dates_overlap` was:

```python
return start_b <= start_a <= end_b
```

This only asks "does the *new* booking's start date fall inside the existing
booking's range?" It completely misses two cases:

- a new booking that starts *before* an existing one but ends inside/after it, and
- a new booking that fully *contains* an existing one.

I replaced it with the standard symmetric, inclusive interval test:

```python
return start_a <= end_b and start_b <= end_a
```

Two inclusive date ranges overlap exactly when each range starts on or before the
other one ends. This catches every overlapping arrangement, not just the one where
the new start lands inside the old range.

## The failure (Task 1)

There is an existing Canon DSLR Camera booking for **Jan 10–15**.

- **Original code wrongly allowed:** a new Canon booking for **Jan 5–12**.
  (`start_a = Jan 5`, and `Jan 10 <= Jan 5` is false, so the old check returned
  "no overlap" — even though Jan 10–12 clearly clashes.)
- **After the fix:** the same Jan 5–12 booking is correctly blocked with a 409.

## Task 2: The PHP 0 booking bug

Rentals are billed *inclusively* — both the start day and the end day count, so a
same-day rental is 1 day. But `rental_days` computed the span *exclusively*:

```python
return (to_date - from_date).days
```

`date - date` yields the number of *whole days between* the two dates, which is one
short of an inclusive count. A same-day booking (`from == to`) came out as `0` days
→ `daily_rate * 0` → **PHP 0.00**, and every other rental was undercharged by
exactly one day.

The fix makes the count inclusive by adding one, and also guards against an inverted
range so a swapped start/end can't silently produce a wrong (or negative) total:

```python
def rental_days(from_date, to_date):
    if to_date < from_date:
        raise ValueError("End date cannot be before start date")
    return (to_date - from_date).days + 1
```

Now a same-day rental correctly charges 1 day, and a Jan 10–15 booking charges 6
days (10, 11, 12, 13, 14, 15) instead of 5. This matches the frontend, which already
did the inclusive `+ 1` in `dayCount`.

## Task 3: No booking equipment under maintenance

Equipment can be marked `"maintenance"` (e.g. the HD Projector), but nothing checked
that status — it could still be listed, shown as available, and booked. The rule
needs to hold everywhere the frontend or API touches equipment, so I enforced it in
four places rather than one:

- **`create_booking`** — the authoritative guard. After resolving the equipment it
  rejects anything not `"available"` with a `409`:

  ```python
  if equipment["status"] != "available":
      return jsonify({
          "error": f"{equipment['name']} is not available for booking"
      }), 409
  ```

- **`/api/availability`** — skips non-available items so maintenance equipment never
  appears in the availability results (`if item["status"] != "available": continue`).

- **`/api/equipment` (`list_equipment`)** — filters out maintenance items from the
  listing, so the booking dropdown never offers them.

- **Frontend `loadEquipment`** — also filters to `status === "available"` before
  building the `<select>`, so a maintenance item can't be selected in the UI.

The backend `create_booking` check is the one that actually enforces the rule; the
availability, listing, and frontend filters keep maintenance equipment out of sight
so the UI stays consistent with what the server will accept.

## Task 4: The frontend price bug

`updateTotal()` reads *both* the start and end date and recomputes the total, but it
was only wired to run on three events — the equipment dropdown, the start-date input,
and the Create-booking button:

```javascript
document.getElementById("equipment").addEventListener("change", updateTotal);
document.getElementById("from").addEventListener("change", updateTotal);
document.getElementById("book").addEventListener("click", createBooking);
```

There was no listener on the end-date (`#to`) input, so changing the end date alone
never recomputed the total. It only *appeared* to update "sometimes": because
`updateTotal()` always reads the current end-date value, touching the start date or
equipment afterward would fire the handler and pick up the already-changed end date.

The fix adds the missing listener, mirroring the start-date one (`change` is the
right event for `<input type="date">`):

```javascript
document.getElementById("to").addEventListener("change", updateTotal);
```

Now editing the end date recomputes the total immediately, on its own.

## AI use

I used an AI coding assistant to help read the codebase quickly, identify the four
bugs, and draft the fixes. Specifically it helped with:

- spotting the one-sided overlap comparison and confirming the correct inclusive
  interval formula;
- catching the missing `change` listener on the end-date input in the frontend;
- drafting this write-up.

How I checked the output was correct:

- I reproduced each bug against the app before changing anything (e.g. booking
  Jan 5–12 over the existing Jan 10–15 booking, and a same-day rental returning
  PHP 0.00).
- I re-ran the same cases after each fix, plus boundary cases (adjacent
  non-overlapping ranges, swapped start/end dates, and the HD Projector under
  maintenance) to confirm the fixes behave correctly and didn't over-block.
- I read every changed line myself rather than accepting suggestions blindly.