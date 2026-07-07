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

### Issue #3 — The same song keeps showing up twice in search

**How you reproduced it:** In `flask shell`, I created a test song with 3 tags and searched for it using `search_songs()`. Instead of returning one result, the same song showed up multiple times, once for each tag.

**How you found the root cause:** I traced the flow from the `/search` endpoint in `routes/songs.py` to `search_service.search_songs()`. Looking through the query, I found it was doing an `outerjoin` on the `song_tags` association table without removing duplicate results.

**The root cause:** Since `song_tags` is a many-to-many table, joining it creates one row per matching tag. So if a song has 3 tags, the query returns 3 rows for that same song. Because the query wasn't deduplicating the results, `search_songs()` returned the same song multiple times.

**Your fix and side-effect check:** I added `.distinct()` to the query so SQLAlchemy only returns each `Song` once, even if it matches multiple tags. I tested it with a song that had 3 tags and it now only appears once. I also checked songs with 0 and 1 tags to make sure they still return correctly.

### Issue #4 — I got notified when a friend added my song to a playlist but not when they rated it

**How you reproduced it:** In `flask shell`, I called `rate_song()` on a song that wasn't owned by the person rating it. Before and after rating the song, I checked the sharer's notifications using `get_notifications()`. The notification count didn't change, even though the rating was saved successfully.

**How you found the root cause:** I compared the code in `add_to_playlist()` and `rate_song()` side by side in `notification_service.py`. The playlist flow creates a notification after adding the song (as long as the user isn't notifying themselves), but the rating flow didn't have anything similar.

**The root cause:** The notification logic was simply missing from `rate_song()`. The function saved the rating and committed it to the database, but it never called `create_notification()`. So the feature wasn't broken because of a bad condition or typo, it was just never implemented in the rating flow.

**Your fix and side-effect check:** I added a `create_notification()` call to `rate_song()` after the rating is saved, following the same logic as `add_to_playlist()`, so users don't get notified about their own actions. I verified that the song's sharer now gets a notification when someone else rates their song, and I also confirmed that playlist notifications still work exactly as before.

