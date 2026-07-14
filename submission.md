# Project 5: Mixtape Bug Fixes

## 1. AI Usage

### AI Usage Instance 1

**What I asked AI:**  
I asked AI to help me understand the main files in the codebase, especially the route files, service files, and models.

**What AI helped me understand:**  
AI helped me understand that the app follows a route-service-model structure. The routes receive API requests and return JSON, the services contain most of the business logic, and the models define the database tables and relationships. AI also helped me trace the listening streak flow from `routes/songs.py` to `services/streak_service.py` and then to the `User` and `ListeningEvent` models.

**What I verified, changed, or rejected:**  
I verified the explanation by opening `app.py`, `models.py`, the route files, and the service files myself. I used the AI explanation as a guide, but I checked the actual function names, route paths, and model fields in the code before adding the codebase map.


### AI Usage Instance 2

**What I asked AI:**  
I asked AI how to run specific pytest tests and how to interpret the failures for the streak, playlist, and search bugs.

**What AI helped me understand:**  
AI helped me run focused tests like `test_streak_increments_on_sunday`, `test_playlist_returns_all_songs`, and `test_search_no_duplicates_multi_tag_song`. It helped me connect each failing assertion to the likely service-layer cause: the Sunday streak condition, the playlist slice that dropped the last song, and the search query returning duplicate rows for multi-tag songs.

**What I verified, changed, or rejected:**  
I verified the results by running the tests myself and inspecting the relevant service files. I accepted the fixes that matched the test evidence, such as removing the Sunday exclusion in the streak logic, returning the full playlist song list, and deduplicating search results. I did not rely only on AI's explanation; I checked the failing tests and code paths directly.


## 2. Codebase Map

### Application Entry Point

`app.py` is the main Flask setup file. It creates the application with `create_app()`, configures the database, initializes SQLAlchemy, registers the route blueprints, and creates the database tables.

The blueprints are registered with these URL prefixes:

- `/songs`
- `/playlists`
- `/users`
- `/feed`

### Database Models

`models.py` defines the database tables and relationships used by the app.

- `User`: stores user profile information, listening streak data, friendships, ratings, notifications, playlists, and shared songs.
- `Song`: stores song information such as title, artist, album, genre, tags, and the user who shared it.
- `ListeningEvent`: stores each time a user listens to a song.
- `Rating`: stores a user's rating for a song.
- `Playlist`: stores playlist information such as name, creator, creation time, and whether it is collaborative.
- `Notification`: stores notifications sent to users.
- `Tag`: stores labels that can be attached to songs.

The file also includes association tables for many-to-many relationships:

- `friendships`: connects users to friends.
- `song_tags`: connects songs to tags.
- `playlist_entries`: connects playlists to songs and stores playlist order.

### Routes

The `routes/` folder contains the Flask API endpoints. These files handle request data, call service functions, and return JSON responses.

- `routes/songs.py`: handles song search, song details, song ratings, and listening events.
- `routes/playlists.py`: handles playlist creation, playlist details, playlist songs, and adding songs to playlists.
- `routes/users.py`: handles user profiles, streak lookups, notifications, and marking notifications as read.
- `routes/feed.py`: handles the social feed and "Friends Listening Now" endpoints.

### Services

The `services/` folder contains the main business logic. The routes call these service functions instead of doing all the logic directly.

- `services/streak_service.py`: records listening events and updates user listening streaks.
- `services/search_service.py`: searches songs by title or artist and retrieves song details.
- `services/playlist_service.py`: creates playlists and retrieves playlist songs.
- `services/feed_service.py`: builds the friends listening-now feed and activity feed.
- `services/notification_service.py`: creates notifications, handles ratings, adds songs to playlists, and retrieves notification data.

### Other Important Files

- `README.md`: explains the project structure, setup instructions, and open bug issues.
- `requirements.txt`: lists the Python packages needed to run the app.
- `seed_data.py`: fills the database with sample users, songs, tags, playlists, listening events, ratings, and notifications.
- `tests/`: contains automated tests for key features like streaks, search, and playlists.
- `submission.md`: documents AI usage, codebase understanding, and bug fixes.

### Architecture Patterns

The project follows a route-service-model structure:

1. A route receives an HTTP request.
2. The route validates request data.
3. The route calls a service function.
4. The service performs the main app logic.
5. The service reads or writes database records through the models.
6. The route returns a JSON response.

This keeps the route files focused on API behavior and keeps most feature logic in the service layer.

### Feature Data Flow

User streak update flow:

1. A user listens to a song by sending a `POST` request to `/songs/<song_id>/listen`.
2. The `listen()` route in `routes/songs.py` reads the JSON request body and checks for `user_id`.
3. If `user_id` is missing, the route returns a `400` error.
4. If the request is valid, the route calls `record_listening_event(user_id, song_id)` from `services/streak_service.py`.
5. `record_listening_event()` looks up the user in the database using the `User` model.
6. The service creates a new `ListeningEvent` row with the user ID, song ID, and current UTC time.
7. The service calls `update_listening_streak(user, now)`.
8. `update_listening_streak()` compares today's date with `user.last_listened_at`.
9. If the user has never listened before, the streak starts at `1`.
10. If the user already listened today, the streak does not change.
11. If the user listened yesterday, the streak increases by `1`.
12. If the user skipped one or more days, the streak resets to `1`.
13. The service updates `user.last_listened_at` to the current listen time.
14. The database commit saves both the new `ListeningEvent` and the updated `User` streak fields.
15. The route returns the listening event as JSON with a `201` status code.

Main files involved:

- `routes/songs.py`: receives the listen request and calls the streak service.
- `services/streak_service.py`: records the listening event and updates the streak.
- `models.py`: defines the `User` and `ListeningEvent` database models used by the streak flow.


## 3. Bug Fixes

### Issue #1 - My listening streak keeps resetting

#### 1. How I Reproduced It

**Endpoint or function:**  
`services.streak_service.update_listening_streak()`

**Test data or conditions:**  
A test user listens on Saturday, June 15, 2024, and then listens again on Sunday, June 16, 2024.

**Steps:**  

1. Open `tests/test_streaks.py`.
2. Run `test_streak_increments_on_sunday`.
3. Compare the user's streak after the Saturday listen and the Sunday listen.

**Expected result:**  
The streak should increase from `1` to `2` because Saturday and Sunday are consecutive calendar days.

**Actual result:**  
With the bug, the streak stayed at `1` because Sunday was treated like a reset instead of a consecutive day.


#### 2. How I Found the Root Cause

**Files inspected:**  
`routes/songs.py`, `services/streak_service.py`, `tests/test_streaks.py`, and `models.py`.

**Call chain followed:**  
`POST /songs/<song_id>/listen` -> `record_listening_event()` -> `update_listening_streak()`

**Debugging method:**  
I used the failing streak test to trace the date comparison logic. The important values were `last_date`, `today`, `days_since_last`, and `today.weekday()`.

**Evidence that confirmed the cause:**  
The buggy condition checked for a consecutive day but excluded Sunday:

```python
elif days_since_last == 1 and today.weekday() != 6:
```

In Python, `weekday() == 6` means Sunday. That meant a Saturday-to-Sunday streak reset even though the dates were consecutive.


#### 3. Root Cause

The streak logic was based on both the number of days since the last listen and the day of the week. The app should only care whether the user listened on consecutive calendar days. The extra Sunday check caused the streak to reset at the start of a new week.


#### 4. Fix

**File changed:**  
`services/streak_service.py`

**Change made:**  
Removed the Sunday exclusion from the consecutive-day condition.

Changed:

```python
elif days_since_last == 1 and today.weekday() != 6:
```

to:

```python
elif days_since_last == 1:
```

**Why this fixes the problem:**  
Now any listen that happens exactly one calendar day after the previous listen increments the streak, including Saturday to Sunday.


#### 5. Side-Effect Checks

I ran the streak tests to check the surrounding behavior:

- A new user starts at streak `1`.
- Listening on two consecutive days increments the streak.
- Listening twice on the same day does not double-count.
- Skipping a day resets the streak.
- Listening on Saturday and then Sunday increments the streak.


### Issue #5 - The last song in a playlist never shows up

#### 1. How I Reproduced It

**Endpoint or function:**  
`services.playlist_service.get_playlist_songs()`

**Test data or conditions:**  
The test creates a playlist with five songs: `Track 1`, `Track 2`, `Track 3`, `Track 4`, and `Track 5`.

**Steps:**  

1. Open `tests/test_playlists.py`.
2. Run `test_playlist_returns_all_songs`.
3. Check how many songs `get_playlist_songs()` returns for the seeded playlist.

**Expected result:**  
The playlist should return all five songs.

**Actual result:**  
With the bug, the playlist returned only four songs. The final song was missing.

#### 2. How I Found the Root Cause

**Files inspected:**  
`routes/playlists.py`, `services/playlist_service.py`, `tests/test_playlists.py`, and `models.py`.

**Call chain followed:**  
`GET /playlists/<playlist_id>/songs` -> `get_songs()` -> `get_playlist_songs()`

**Debugging method:**  
I ran the focused playlist test and used the assertion failure to compare the expected song count with the returned song count.

**Evidence that confirmed the cause:**  
The failing test expected `len(songs) == 5`, but the service returned `4`. The query retrieved the playlist songs, but the return statement sliced off the last item:

```python
return [song.to_dict() for song in songs[:-1]]
```

#### 3. Root Cause

The service used `songs[:-1]`, which means "all songs except the last one." Because of that slice, the last song in every non-empty playlist was removed before the data was returned.

#### 4. Fix

**File changed:**  
`services/playlist_service.py`

**Change made:**  
Changed the return statement so it returns the full song list instead of slicing off the final item.

Changed:

```python
return [song.to_dict() for song in songs[:-1]]
```

to:

```python
return [song.to_dict() for song in songs]
```

**Why this fixes the problem:**  
The query already returns the songs in playlist order. Removing the slice keeps every song in the result, including the final song.

#### 5. Side-Effect Checks

I checked the related playlist behavior:

- A playlist with five songs returns five songs.
- Songs are still returned in position order.
- An empty playlist still returns an empty list.


### Issue #3 - The same song keeps showing up twice in search

#### 1. How I Reproduced It

**Endpoint or function:**  
`services.search_service.search_songs()`

**Test data or conditions:**  
The test creates a song named `Crown Heights Anthem` with three tags: `rap`, `hip-hop`, and `boom bap`.

**Steps:**  

1. Open `tests/test_search.py`.
2. Run `test_search_no_duplicates_multi_tag_song`.
3. Search for `Crown Heights`.
4. Count how many times `Crown Heights Anthem` appears in the results.

**Expected result:**  
The song should appear exactly once in the search results.

**Actual result:**  
With the bug, the song appeared three times because it had three tag rows joined to it.

#### 2. How I Found the Root Cause

**Files inspected:**  
`routes/songs.py`, `services/search_service.py`, `tests/test_search.py`, and `models.py`.

**Call chain followed:**  
`GET /songs/search?q=Crown Heights` -> `search()` -> `search_songs()`

**Debugging method:**  
I ran the focused search test and checked how many matching dictionaries were returned for the same song title.

**Evidence that confirmed the cause:**  
The test expected one result, but the buggy query returned one row per tag association. A multi-tag song could appear once for each matching joined tag row.

#### 3. Root Cause

The search query joined songs to the `song_tags` association table. When a song had multiple tags, the join could produce multiple rows for the same song. Without deduplicating the results, the same song appeared more than once in the API response.

#### 4. Fix

**File changed:**  
`services/search_service.py`

**Change made:**  
Updated the search query so it returns distinct `Song` records instead of duplicate joined rows.

The fixed version queries `Song` and uses `.distinct()`:

```python
results = (
    db.session.query(Song)
    .outerjoin(song_tags, Song.id == song_tags.c.song_id)
    .filter(
        db.or_(
            Song.title.ilike(f"%{query}%"),
            Song.artist.ilike(f"%{query}%"),
        )
    )
    .distinct()
    .all()
)
```

**Why this fixes the problem:**  
`.distinct()` removes duplicate song rows caused by the tag join, so each matching song appears once even if it has multiple tags.

#### 5. Side-Effect Checks

I checked the related search behavior:

- A basic title or artist search still returns matching songs.
- A song with no tags still appears once.
- A song with one tag still appears once.
- A song with multiple tags appears once instead of once per tag.
- A query with no matches still returns an empty list.
