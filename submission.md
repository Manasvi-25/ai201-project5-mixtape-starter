# Mixtape Bug Hunt — Submission

## AI Usage

I used Claude throughout this project mostly to verify my debugging process, not to find the bugs for me or write the fixes.

**Navigation:** Before starting the bugs, I used Claude to get a high-level overview of the codebase. It helped me understand how the models, routes, and services were organized so I knew where to start tracing each issue.

**Tracing and hypothesis-checking:** For every issue, I first traced the code myself from the reported endpoint through the relevant service functions until I had a theory about what was causing the bug. Only after that did I use Claude to sanity-check my reasoning, explain any confusing code, or confirm that I hadn't missed another execution path. For Issue #1, I traced the streak update logic myself and noticed the Sunday `weekday()` check before using Claude to confirm my understanding of how the condition behaved. For Issue #3, I initially thought I had found the problem, but my `flask shell` tests didn't match what I expected. After digging through the database and recreating clean test data, I realized the confusing results were caused by leftover test data from an earlier shell session, and only then used Claude to confirm that the `outerjoin` without `.distinct()` was the actual root cause.

**What I verified myself:** I reproduced every bug, implemented every fix, and ran all the verification steps myself in `flask shell`. After each change, I also checked that the normal behavior still worked so I wasn't introducing new bugs while fixing the original one.

**Where AI output could have been misleading:** During Issue #3, an early explanation about the SQL join sounded reasonable, but it didn't line up with my test results. Instead of assuming the explanation was correct, I kept testing until I found the real issue was my own leftover test data. That was a good reminder that AI explanations still need to be verified against the actual code and data.

## Codebase Map

**app.py** — Creates the Flask app, initializes the database, and registers all the blueprints.

**models.py** — Defines all the database models (`User`, `Song`, `Tag`, `ListeningEvent`, `Rating`, `Playlist`, and `Notification`) along with the association tables (`friendships`, `song_tags`, and `playlist_entries`) that handle many-to-many relationships.

**routes/** — Contains one blueprint per feature (`songs.py`, `playlists.py`, `users.py`, `feed.py`). These files mostly parse request data, call the appropriate service function, and return a JSON response. Almost no business logic lives here.

**services/** — This is where the application's core logic lives.
- `streak_service.py` handles listening streak calculations.
- `feed_service.py` builds the Friends Listening Now and activity feeds.
- `search_service.py` handles song search and filtering.
- `notification_service.py` manages notifications, song ratings, and playlist notification logic.
- `playlist_service.py` creates playlists and returns songs in playlist order.

**Data flow — User rates a song:**

`POST /songs/<song_id>/rate` (`routes/songs.py`) → `notification_service.rate_song()` → validate the rating → create or update the `Rating` record → commit the change → call `create_notification()` (if the rater isn't the song's sharer) → return the updated rating as JSON.

**Pattern noticed:** Every route follows the same pattern: parse the request, call a service function, and return the response. The actual business logic lives almost entirely in the `services/` layer, which is also where every bug I investigated ended up being.

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

### Issue #5 — The last song in a playlist never shows up

**How you reproduced it:** In `flask shell`, I checked the raw `playlist_entries` table for an existing playlist and confirmed it had 7 songs (positions 1–7). Then I called `get_playlist_songs()` on the same playlist, and it only returned 6 songs. The song in the last position was always missing.

**How you found the root cause:** I traced the flow from the `/playlists/<id>/songs` endpoint in `routes/playlists.py` to `playlist_service.get_playlist_songs()`. After looking through the query, I noticed the final return statement was slicing the list with `songs[:-1]`.

**The root cause:** The function was returning `songs[:-1]`, which removes the last item from the list every time. Since the songs are already sorted by position, the last item is always the newest song in the playlist. That explains why the most recently added song never appeared, and why adding another song made the previously missing one show up.

**Your fix and side-effect check:** I removed the unnecessary `[:-1]` slice so the function returns the full list of songs. I verified that a playlist with 7 entries now returns all 7 songs in the correct order.