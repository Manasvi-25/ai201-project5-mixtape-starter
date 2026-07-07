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

### Issue #2 — Friends Listening Now shows people from yesterday

**How you reproduced it:** In `flask shell`, I created a `ListeningEvent` for a friend with a timestamp 10 hours in the past (an 11pm listen checked at 9am the next day). When I called `get_friends_listening_now()`, the friend still showed up in the feed even though the listen was from yesterday.

**How you found the root cause:** I traced the flow from the `/listening-now` endpoint in `routes/feed.py` to `feed_service.get_friends_listening_now()`. There, I found that the feed was using `RECENT_THRESHOLD = timedelta(hours=24)` to decide which events counted as "recent."

**The root cause:** The cutoff was calculated as `datetime.now(timezone.utc) - timedelta(hours=24)`, which creates a rolling 24-hour window instead of resetting at the start of a new day. Because of that, an 11pm listen from yesterday stayed in the feed until 11pm today, exactly matching the reported bug.

**Your fix and side-effect check:** I changed the cutoff to reset at midnight using `datetime.now(timezone.utc).replace(hour=0, minute=0, second=0, microsecond=0)`, so only events from the current day are considered "listening now." I verified that events from earlier today still appear, while events from 11pm the previous day are correctly filtered out.