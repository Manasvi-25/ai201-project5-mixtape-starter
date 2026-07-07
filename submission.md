# Mixtape Bug Hunt — Submission

## AI Usage

(To be completed in Milestone 4. I'll describe how I used AI to navigate the codebase, trace bugs, and verify fixes, along with any suggestions I accepted or rejected.)

## Codebase Map

(To be completed. Will include the main files, what each one is responsible for, and a data flow trace for at least one feature.)

## Root Cause Analysis

### Issue #1 — My listening streak keeps resetting

**How you reproduced it:** Using `flask shell`, I manually set a user's `last_listened_at` to Saturday, July 4, 2026 with `listening_streak = 12`. I then called `update_listening_streak(user, fake_now)` with `fake_now` set to Sunday, July 5, 2026. Instead of increasing the streak to 13, it reset to 1.

**How you found the root cause:** I traced the flow from the `/streak` endpoint in `routes/users.py`, to `streak_service.get_streak()`, and then to `update_listening_streak()`, which is called from `record_listening_event()`. From there, I checked the update logic line by line until I found the condition causing the reset.

**The root cause:** The increment condition is `days_since_last == 1 and today.weekday() != 6`. Since `weekday()` returns `6` for Sunday, the code refuses to continue the streak if the next day happens to be Sunday. It skips the increment and falls into the reset branch instead. There's no reason for Sundays to be treated differently here, so this extra check is causing valid streaks to reset.

**Your fix and side-effect check:** I removed the `and today.weekday() != 6` part from the increment condition in `update_listening_streak()`, so now the streak increments whenever exactly one day has passed, regardless of the day of the week. I tested the original bug and the streak now correctly goes from 12 to 13 on Sunday. I also checked that a normal consecutive day still increments properly (5 → 6), and that skipping multiple days still resets the streak back to 1, so the rest of the logic still works as expected.