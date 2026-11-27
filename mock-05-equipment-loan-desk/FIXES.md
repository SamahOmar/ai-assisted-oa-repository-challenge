
# Equipment Loan Desk — Fixes
## Test 1: `test_create_loan`

### Bug 1: Incorrect integer validation
**Failure:** `AssertionError: 201 != 400`

**Root cause:**

The quantity validator used `isinstance(quantity, int)`, which also accepts booleans because Python's `bool` type is a subclass of `int`.

An existing helper already handled this distinction:

```python
def _is_int(value):
    return isinstance(value, int) and not isinstance(value, bool)
```

**Fix:**

Changed:

```python
if not isinstance(quantity, int) or quantity < 1:
```

To:

```python
if not _is_int(quantity) or quantity < 1:
    raise ValidationFailed("quantity must be a whole number of at least 1")
```

**Classification:** Logical / Validation error.


### Bug 2: Incorrect loan duration calculation

**Root cause:**

The `loan_days()` function used `max()` even though its docstring and README specified that a loan containing multiple equipment categories must use the shortest permitted loan period.

**Fix:**

Changed:

```python
return max(LOAN_PERIOD_DAYS[category] for category in categories)
```

To:

```python
return min(LOAN_PERIOD_DAYS[category] for category in categories)
```

**Classification:** README contract violation / Logical error.

### Bug 3: Equipment availability checked after loan creation

**Root cause:**

The implementation created the loan before checking whether the requested equipment was available.

This caused an unavailable loan to be created instead of returning HTTP 409 Conflict.

**Fix:**

Reordered the operations:

- Before: Create the loan → Check equipment availability.
- After: Check equipment availability → Create the loan.

**Classification:** Logical error / Incorrect operation order.

---

## Test 2: `test_return_loan`

### Bug 1: Incorrect argument passed to the late-fee calculation

**Root cause:**

The fee calculation received `due_on` instead of the actual number of late days.

**Fix:**

Passed the calculated number of late days to the fee calculation method instead of the due date.

### Bug 2: Incorrect late-fee calculation

**Root cause:**

The implementation calculated fees using all late days instead of excluding the permitted grace period (`mercy_days`).

**Fix:**

Changed the calculation to use:

```python
late_days - mercy_days
```

The fee should only apply to days beyond the grace period.

**Classification:** Logical errors / Business rule violations.

---

## Test 3: `test_extend_and_cancel`

**Failure:** `AssertionError: 409 != 201`

Loan creation returned HTTP 409 Conflict instead of HTTP 201 Created.

**Investigation:**

I initially inspected:

- `cancel_loan()` in `loans.py`.
- `extend_loan()`.
- `create_loan()`.

The actual issue was in `available_units()`.

**Root cause:**

The availability calculation excluded only returned loans, but cancelled loans were still counted as occupying inventory.

**Fix:**

Changed:

```python
.exclude(loan__status="returned")
```

To:

```python
.exclude(loan__status__in=["returned", "cancelled"])
```

**Classification:** Logical error.

**Lesson:** A test's name does not necessarily indicate where the bug is. Read the failed assertion carefully. The actual failure message often provides a much better starting point for debugging than the test name itself.

---

## Test 4: `test_list_filters`

**Failure:**

```python
AssertionError: Lists differ: [2, 3] != [3]
```

The equipment filter worked independently, but combining it with the status filter produced incorrect results.

**Investigation:**

I checked `filter_loans()` in `queries.py` and compared the behavior of individual filters with their combined behavior.

**Root cause:**

The implementation created a new QuerySet using `Loan.objects.filter()`, replacing the existing QuerySet and discarding previously applied filters.

**Fix:**

Use the existing QuerySet:

```python
loans.filter(...)
```

Instead of:

```python
Loan.objects.filter(...)
```

**Classification:** Logical error / QuerySet handling.

---

## Test 5: `test_sorting_and_paging`

**Failure:**

```python
self.assertEqual(first["pages"], 3)
AssertionError: 2 != 3
```

**Root cause:**

The pages calculation in `list_page()` used floor division (`//`), while the README specified that the total number of pages must be rounded up.

**Fix:**

Changed:

```python
pages = total // params["per_page"]
```

To:

```python
pages = (total + params["per_page"] - 1) // params["per_page"]
```

**Classification:** README contract violation / Boundary condition.
---

## Test 6: `test_borrower_summary`

**Failure:**

The `late_returns` count was 2 instead of the expected 1.

**Investigation:**

I opened `reports.py` and inspected the `row["late_returns"]` calculation inside `borrower_summary()`.

The README defined `late_returns` as the number of loans returned with `fee_cents` greater than zero.

**Root cause:**

The implementation checked whether the return date was later than the due date instead of checking whether a late fee had actually been charged.

**Fix:**

Changed:

```python
if loan.status == "returned" and loan.returned_on > loan.due_on:
    row["late_returns"] += 1
```

To:

```python
if loan.status == "returned" and loan.fee_cents > 0:
    row["late_returns"] += 1
```

**Classification:** README contract violation / Logical error.

---



# Summary 

| Test | Bugs discovered and fixes | Classification |
|---|---|---|
| `test_create_loan` | Quantity validation accepted booleans because it used `isinstance()` directly; used `max()` instead of `min()` for loan duration; checked availability after creating the loan. | Logic / README contract |
| `test_return_loan` | Passed the wrong value to late-fee calculation; calculated fees using all late days instead of days beyond the grace period. | Logic / README contract |
| `test_extend_and_cancel` | Cancelled loans were still counted as occupying inventory. | Logic |
| `test_list_filters` | A new QuerySet replaced the existing one, causing previous filters to disappear. | Logic |
| `test_sorting_and_paging` | Used floor division instead of ceiling division when calculating total pages. | README contract |
| `test_borrower_summary` | Counted loans as late based on dates instead of the documented positive-fee condition. | README contract |
