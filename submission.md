# Codebase Map

models.py defines 7 SQLAlchemy models: User, Tag, Song, ListeningEvent, Rating, Playlist, and Notification. Song-to-playlist membership isn't a model at all — it's a plain association table, playlist_entries, with extra columns (position, added_by, added_at) beyond the two foreign keys. Similarly, friendships (User-to-User) and song_tags (Song-to-Tag) are association tables, not models.

Routes (routes/) are thin: each one parses the request, calls exactly one service function, and formats the response as JSON.
- routes/songs.py: GET /songs/search, GET /songs/<id>, POST /songs/<id>/rate, POST /songs/<id>/listen
- routes/playlists.py: POST /playlists/, GET /playlists/<id>, GET /playlists/<id>/songs, POST /playlists/<id>/songs
- routes/users.py: GET /users/<id>, GET /users/<id>/streak, GET /users/<id>/notifications, POST /users/notifications/<id>/read
- routes/feed.py: GET /feed/<id>/listening-now, GET /feed/<id>/activity

Services (services/) hold all the business logic:
- streak_service.py: record_listening_event() writes a ListeningEvent row, then calls update_listening_streak() to increment/reset the user's streak based on calendar-day gaps.
- feed_service.py: get_friends_listening_now() and get_activity_feed() both read from the same ListeningEvent table, filtered to the current user's friends — one restricts to the last 24 hours and dedupes to one song per friend, the other just returns the most recent N events unfiltered.
- search_service.py: search_songs() does a case-insensitive LIKE match on title/artist, outer-joined to song_tags to pull in tags.
- playlist_service.py: create_playlist(), get_playlist(), get_playlist_songs() (orders songs by their position column in playlist_entries), and get_user_playlists().
- notification_service.py: create_notification() is the shared primitive; add_to_playlist() calls it to notify a song's original sharer when someone else adds their song to a playlist; rate_song() saves a Rating row but does not call create_notification() at all.

Data flow — user rates a song: POST /songs/<id>/rate in routes/songs.py calls notification_service.rate_song(). That function creates (or updates, if one already exists) a Rating record — a separate model with a unique constraint on (user_id, song_id) — but never creates a Notification. Compare this to add_to_playlist(), which does call create_notification() after adding a song. This asymmetry is Issue #4: the sharer is notified when their song is added to a playlist, but not when it's rated.

Data flow — user views a playlist: GET /playlists/<id>/songs in routes/playlists.py calls playlist_service.get_playlist_songs(), which queries Song joined to playlist_entries, ordered by position, then slices the result with songs[:-1] before returning — which drops the last song in the playlist. This is Issue #5.

Data flow — a song gets into a friend's feed: there's no explicit "add to feed" step. POST /songs/<id>/listen calls streak_service.record_listening_event(), which just inserts a ListeningEvent row. Later, GET /feed/<id>/listening-now calls feed_service.get_friends_listening_now(), which queries those same ListeningEvent rows for the user's friends. The feed is entirely derived at read time from listening history — there's no separate feed table.

Pattern I noticed: every route delegates immediately to a service function. The routes do input parsing and response formatting; all business logic lives in services/. A second pattern: services never validate cross-cutting concerns themselves (e.g. friendship checks) — feed_service filters by user.friends directly in each query rather than through a shared helper, which is worth watching for duplicated/inconsistent logic across get_friends_listening_now() and get_activity_feed().

# Reproducing Bugs:

## Bug #2: Friends Listening Now shows people from yesterday
I ran:
source .venv/bin/activate && python -c "
from app import create_app, db
from models import User, ListeningEvent, Song
from datetime import datetime, timezone, timedelta

app = create_app()
with app.app_context():
    aaliya = User.query.filter_by(username='aaliya').first()
    song = Song.query.first()

    today_midnight = datetime.now(timezone.utc).replace(hour=0, minute=0, second=0, microsecond=0)
    yesterday_2350 = today_midnight - timedelta(minutes=10)

    event = ListeningEvent(user_id=aaliya.id, song_id=song.id, listened_at=yesterday_2350)
    db.session.add(event)
    db.session.commit()

    print('aaliya id:', aaliya.id)
    print('kenji id:', User.query.filter_by(username=\"kenji\").first().id)
    print('inserted listened_at:', yesterday_2350, '(calendar date: yesterday)')
    print('age in hours:', (datetime.now(timezone.utc) - yesterday_2350).total_seconds() / 3600)
"

Then I ran: curl -s http://127.0.0.1:5000/feed/73daacee-8381-4a8b-bcd5-61f9f5ebc19f/listening-now | python -m json.tool
 because Kenji is Aaliya's friend. I discovered that feed_service.get_friends_listening_now() uses a flat rolling window 
 This only checks "is it less than 24 hours old," not "did this happen today." An event from 23:50 last night is ~5 hours old (well under 24h), so it passes the filter — but it's from a different calendar date, which is exactly the reported symptom: "Friends Listening Now shows people from yesterday."

This is inconsistent with how streak_service.py handles the same kind of problem — it explicitly compares .date() values (calendar days) rather than raw hour deltas. feed_service.py should likely do the same: filter by "listened today" (i.e. listened_at.date() == today) rather than "listened within the last 24 hours."

To fix the error I removed the line that says:
RECENT_THRESHOLD = timedelta(hours=24)
Then I changed the cutoff to be:   
cutoff = datetime.now(timezone.utc).replace(hour=0, minute=0, second=0, microsecond=0)
Before the cutoff was a rolling window. This means we want it to mean midnight today, so anything before that, even if it is less than 24 hour old...gets excluded. 


# Bug 3: The same song keeps showing up twice in search

I ran:
source .venv/bin/activate && python -c "
from app import create_app, db
from models import Song, song_tags

app = create_app()
with app.app_context():
    stmt = (
        db.session.query(Song.id, Song.title)
        .outerjoin(song_tags, Song.id == song_tags.c.song_id)
        .filter(Song.title.ilike('%Crown Heights%'))
        .statement
    )
    raw_rows = db.session.execute(stmt).fetchall()
    for row in raw_rows:
        print(row)
    print('raw row count:', len(raw_rows))
"

This queries the exact same join search_songs() uses in services/search_service.py, but selects raw columns (Song.id, Song.title) instead of the full Song entity. The result was:
('d2386bfc-2fc7-4cee-b323-596504383fdf', 'Crown Heights Anthem')
('d2386bfc-2fc7-4cee-b323-596504383fdf', 'Crown Heights Anthem')
('d2386bfc-2fc7-4cee-b323-596504383fdf', 'Crown Heights Anthem')
raw row count: 3

"Crown Heights Anthem" has 3 tags (rap, hip-hop, boom bap) in song_tags. The outerjoin in search_songs() fans out one row per matching tag, so the raw SQL genuinely returns the same song 3 times — one row per tag row it joins against.

Interestingly, calling search_songs('Crown Heights') directly (or hitting GET /songs/search?q=Crown, or running pytest tests/test_search.py) does NOT show this duplication — all come back with exactly 1 result. That's because search_songs() calls db.session.query(Song) — the legacy SQLAlchemy Query API — selecting only the full Song entity with no extra columns. That specific pattern silently deduplicates by primary key when materializing .all(), which is why the duplicate rows never reach the caller today. The bug is masked, not fixed: the query itself is still wrong (it's expressing "one row per matching song-tag pair," not "one row per matching song"), and it happens to come out right only because of an ORM implementation detail this code doesn't explicitly rely on.

To fix it, I added .distinct() to the query in search_service.py so it explicitly returns one row per song regardless of ORM version/behavior:

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
