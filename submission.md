Codebase Map

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