# Bug Report — CoWork API

This report documents all bugs found during review, grouped by module. For each
bug: affected file(s)/function, the business rule it violates, what was wrong,
why it caused incorrect behavior, and how it was fixed.

---

## 1. Auth

### 1.1 Duplicate username not rejected
- **File/function:** `routers/auth.py` — `register()`
- **Rule:** #15 (Registration)
- **Bug:** When a duplicate username was submitted, the endpoint returned
  `201` with the *existing* user's details instead of rejecting the request.
- **Impact:** Duplicate registration attempts appeared to succeed, silently
  returning someone else's user record.
- **Fix:** Raise `AppError(409, "USERNAME_TAKEN", ...)` when a user with the
  same username already exists in the org.

### 1.2 Access token expiration wrong by 60x
- **File/function:** `services/auth.py` — `create_access_token()`
- **Rule:** #8 (Auth)
- **Bug:** `timedelta(minutes=ACCESS_TOKEN_EXPIRE_MINUTES * 60)` multiplied
  minutes by 60 before building the `timedelta`, producing a 900-minute
  lifetime instead of 900 seconds.
- **Impact:** Access tokens stayed valid 60x longer than the spec requires.
- **Fix:** Use `timedelta(minutes=ACCESS_TOKEN_EXPIRE_MINUTES)` so
  `exp − iat` is exactly 900 seconds.

### 1.3 Logout doesn't actually revoke the token
- **File/function:** `services/auth.py` — `revoke_access_token()` /
  `get_token_payload()`
- **Rule:** #8 (Auth)
- **Bug:** Revocation stored the token's `jti`, but the validity check looked
  up `payload["sub"]` (the user id) in the revoked set — a mismatched key.
- **Impact:** Logged-out access tokens continued to work indefinitely.
- **Fix:** Check the revoked set using `payload["jti"]` on both write and
  read paths.

### 1.4 Refresh tokens are reusable
- **File/function:** `routers/auth.py` — `refresh()`
- **Rule:** #8 (Auth)
- **Bug:** New tokens were issued without invalidating the refresh token that
  was presented.
- **Impact:** The same refresh token could be replayed indefinitely instead
  of triggering `401` on reuse.
- **Fix:** Record the presented refresh token's `jti` as used at issuance
  time; reject any later request presenting the same `jti` with `401`.

### 1.5 Race condition on first-time org creation
- **File/function:** `routers/auth.py` — `register()`
- **Rule:** #15 (Registration)
- **Bug:** Two concurrent registrations for the same new `org_name` both
  attempted to insert a new `Organization`, and the unique-name constraint
  caused the loser to fail with an unhandled `IntegrityError`.
- **Impact:** Concurrent org creation could crash the request instead of the
  second caller cleanly joining the org.
- **Fix:** Wrap org creation in try/except `IntegrityError`; on conflict,
  roll back and re-fetch the now-existing org, then continue registration as
  a join.

---

## 2. Bookings — validation

### 2.1 Past start times accepted (grace window)
- **File/function:** `bookings.py` — `create_booking()`
- **Rule:** #2 (Booking price / window)
- **Bug:** `if start <= now - timedelta(seconds=300):` allowed start times up
  to 5 minutes in the past.
- **Impact:** Bookings starting in the past (e.g. 3 minutes ago) were
  accepted, though the spec allows no grace window at all.
- **Fix:** `if start <= now: raise AppError(...)`.

### 2.2 Missing `end_time > start_time` check
- **File/function:** `bookings.py` — `create_booking()`
- **Rule:** #2
- **Bug:** No validation compared `end_time` to `start_time`.
- **Impact:** Requests with `end_time <= start_time` (equal or reversed)
  were processed, risking negative durations/prices.
- **Fix:** Raise `400 INVALID_BOOKING_WINDOW` when `end <= start`.

### 2.3 Missing minimum-duration check
- **File/function:** `bookings.py` — `create_booking()`
- **Rule:** #2
- **Bug:** Only the maximum duration (8h) was validated; there was no lower
  bound.
- **Impact:** Zero- or negative-hour bookings could pass validation.
- **Fix:** Add `if duration_hours < MIN_DURATION_HOURS: raise
  AppError(400, "INVALID_BOOKING_WINDOW", ...)` before the max check.

### 2.4 Overlap detection used `<=` instead of `<`
- **File/function:** `bookings.py` — `_has_conflict()`
- **Rule:** #3 (No double-booking)
- **Bug:** `if b.start_time <= end and start <= b.end_time:` treated
  back-to-back bookings as overlapping.
- **Impact:** A booking ending at 11:00 blocked a new booking starting at
  11:00, though the spec explicitly allows this.
- **Fix:** Use strict inequalities: `if b.start_time < end and start <
  b.end_time:`.

---

## 3. Bookings — concurrency

### 3.1 Conflict check and insert not atomic
- **File/function:** `routers/bookings.py` — `create_booking()`
- **Rule:** #3 (No double-booking), #16 (Liveness)
- **Bug:** The overlap check and the booking insert were separate,
  unsynchronized steps.
- **Impact:** Two concurrent requests for overlapping slots could both pass
  the conflict check before either was committed, producing two overlapping
  `confirmed` bookings.
- **Fix:** Added `_booking_lock` (a `threading.Lock`) around the conflict
  check, quota check, and insert so the whole section runs atomically per
  process.

### 3.2 Quota check not atomic
- **File/function:** `routers/bookings.py` — `create_booking()` /
  `_check_quota()`
- **Rule:** #4 (Booking quota), #16 (Liveness)
- **Bug:** The 3-bookings-per-24h quota check ran outside any lock.
- **Impact:** Concurrent requests from the same user could each pass the
  quota check simultaneously and collectively exceed the limit.
- **Fix:** Moved the quota check inside the same `_booking_lock` used for
  the conflict check, so check-and-create is atomic.

---

## 4. Bookings — listing & retrieval

### 4.1 Pagination offset off-by-one page
- **File/function:** `bookings.py` — `list_bookings()`
- **Rule:** #11 (Pagination & ordering)
- **Bug:** `.offset(page * limit)` instead of `.offset((page - 1) * limit)`.
- **Impact:** `page=1` skipped the first `limit` items entirely.
- **Fix:** `.offset((page - 1) * limit)`.

### 4.2 `limit` query param ignored
- **File/function:** `bookings.py` — `list_bookings()`
- **Rule:** #11
- **Bug:** `.limit(10)` was hardcoded.
- **Impact:** Requests with `limit=5` or `limit=50` always got 10 results.
- **Fix:** `.limit(limit)`.

### 4.3 Wrong sort order
- **File/function:** `bookings.py` — `list_bookings()`
- **Rule:** #11
- **Bug:** Sorted by `Booking.start_time.desc()`.
- **Impact:** Newest-first instead of the required ascending `start_time`
  (ties by ascending `id`).
- **Fix:** `Booking.start_time.asc()`.

### 4.4 Members can read other members' bookings
- **File/function:** `bookings.py` — `get_booking()`
- **Rule:** #10 (Booking visibility)
- **Bug:** The lookup checked booking id + org only, never ownership.
- **Impact:** Any member could fetch another member's booking by guessing/
  knowing its id.
- **Fix:** After fetch, `if user.role != "admin" and booking.user_id !=
  user.id: raise AppError(404, "BOOKING_NOT_FOUND", ...)`.

### 4.5 Wrong `start_time` in response
- **File/function:** `bookings.py` — `get_booking()`
- **Rule:** API contract (Booking schema)
- **Bug:** `response["start_time"] = iso_utc(booking.created_at)` returned
  the creation timestamp instead of the booking's actual start time.
- **Impact:** Clients received an incorrect `start_time` for every booking
  fetched by id.
- **Fix:** Use `booking.start_time` instead of `booking.created_at`.

---

## 5. Bookings — cancellation & refunds

### 5.1 Incorrect refund percentage for <24h notice
- **File/function:** `bookings.py` — `cancel_booking()`
- **Rule:** #6 (Cancellation refund policy)
- **Bug:** The `else` branch (notice < 24h) set `refund_percent = 50`.
- **Impact:** Late cancellations wrongly received a 50% refund instead of 0%.
- **Fix:** `refund_percent = 0` for the < 24h case.

### 5.2 Refund rounding used banker's rounding
- **File/function:** `bookings.py` — `cancel_booking()`
- **Rule:** #6
- **Bug:** Python's built-in `round()` rounds half-to-even, not half-up.
- **Impact:** E.g. 50% of 1001 cents (500.5) rounded down to 500 instead of
  the required 501.
- **Fix:** Use `Decimal(...).quantize(Decimal("1"), rounding=ROUND_HALF_UP)`.

### 5.3 Refund amount could diverge between response and RefundLog
- **File/function:** `bookings.py` — `cancel_booking()`; `services/refunds.py`
  — `log_refund()`
- **Rule:** #6
- **Bug:** The refund amount was calculated twice — once for the API
  response, and again inside `log_refund()` (using float math instead of
  `Decimal`) — so the two values could disagree.
- **Impact:** The amount in the cancel response wasn't guaranteed to match
  the `RefundLog` entry, violating the "amount returned equals amount
  stored" requirement.
- **Fix:** Calculate `refund_amount_cents` once (with `Decimal` +
  `ROUND_HALF_UP`) in `cancel_booking()`, and pass that single value into
  `log_refund(db, booking, amount_cents)`, which now just stores it instead
  of recomputing it.

---

## 6. Rooms

### 6.1 Room stats race condition
- **File/function:** `services/stats.py` — `record_create()`,
  `record_cancel()`, `get()`
- **Rule:** #14 (Room stats), #16 (Liveness)
- **Bug:** Stats were kept in a shared in-memory dict updated without any
  synchronization.
- **Impact:** Concurrent creates/cancels could overwrite each other's
  updates, leaving `/rooms/{id}/stats` counts and revenue inconsistent with
  the actual bookings.
- **Fix:** Added a module-level `_stats_lock` (`threading.Lock`) and wrapped
  all three functions' read/update logic in `with _stats_lock:`.

---

## 7. Admin

### 7.1 Export leaks cross-org bookings
- **File/function:** `services/export.py` — `generate_export()`,
  `fetch_bookings_raw()`
- **Rule:** #9 (Multi-tenancy)
- **Bug:** When `include_all=True` with a `room_id`, `fetch_bookings_raw()`
  filtered only by `room_id`, without checking that the room belongs to the
  caller's org.
- **Impact:** An admin could export another organization's bookings simply
  by supplying that org's `room_id`.
- **Fix:** `fetch_bookings_raw(db, org_id, room_id)` now filters by both
  `Room.org_id == org_id` and `Booking.room_id == room_id`; updated the call
  site in `generate_export()` to pass `org_id`.

---

## 8. Shared services

### 8.1 Reference code collisions under concurrency
- **File/function:** `services/reference.py` — `next_reference_code()`
- **Rule:** #7 (Reference codes)
- **Bug:** The shared counter was read and incremented without
  synchronization.
- **Impact:** Concurrent booking creation could read the same counter value
  before either write landed, producing duplicate `reference_code`s.
- **Fix:** Added `_counter_lock` (`threading.Lock`) around the read-increment
  section so each call atomically claims a unique value.

### 8.2 Rate limiter bucket race condition
- **File/function:** `services/ratelimit.py` — `record_and_check()`
- **Rule:** #5 (Rate limit)
- **Bug:** The per-user request bucket (trim → append → store → check) was
  updated without synchronization.
- **Impact:** Concurrent requests could lose or overwrite bucket updates,
  causing incorrect counts and letting users exceed (or get falsely blocked
  by) the 20-requests/60s limit.
- **Fix:** Added `_bucket_lock` (`threading.Lock`) around the full
  trim/append/store/check sequence so it executes atomically per call.

### 8.3 Notification deadlock from inconsistent lock ordering
- **File/function:** `services/notifications.py` — `notify_created()`,
  `notify_cancelled()`
- **Rule:** #16 (Liveness)
- **Bug:** `notify_created()` and `notify_cancelled()` acquired
  `_email_lock` and `_audit_lock` in different orders.
- **Impact:** Concurrent create/cancel calls could each hold one lock while
  waiting on the other, deadlocking and hanging the service.
- **Fix:** Reordered `notify_cancelled()` to acquire `_email_lock` before
  `_audit_lock`, matching `notify_created()`, so lock order is always
  consistent.

---
8.4 Cancellation race condition creates duplicate RefundLog entries
File/function: routers/bookings.py — cancel_booking()
Rule: #6 (Cancellation refund policy)
Bug: cancel_booking() had no lock protection around the status check, refund calculation, and RefundLog creation. Two concurrent cancellation requests could both pass the status == "cancelled" check and create separate refund records.
Impact: A single cancelled booking could have multiple RefundLog entries and duplicate refunds, violating the requirement that a cancelled booking has exactly one RefundLog entry and that concurrent cancellations are handled safely.
Fix: Wrapped the cancellation process inside the existing _booking_lock, moving the booking status update, refund creation, and database commit inside the lock. This ensures only one cancellation request can process a booking at a time, while later requests receive 409 ALREADY_CANCELLED.


Bug 24

File(s):
app/timeutils.py (parse_input_datetime())

README Rule:
Business Rule 1 – Datetimes

Bug:
UTC offset datetimes were handled by removing the timezone information using dt.replace(tzinfo=None) instead of converting the datetime to UTC first.

Why it caused incorrect behavior:
Datetime values with offsets were stored with the wrong time. For example, 10:00+05:00 was stored as 10:00 instead of being converted to 05:00 UTC, causing incorrect booking conflicts, quota checks, availability, and reports.

How it was fixed:
Replaced dt.replace(tzinfo=None) with dt.astimezone(timezone.utc).replace(tzinfo=None) so offset-aware datetimes are converted to UTC before storage.

