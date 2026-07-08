AI usage section:

I had Gemini explain to me some snippets or functions to understand the syntax and what some functions returned. I also used Gemini to explain error statements. For example, I asked what does return [song.to_dict() for song in songs[:-1]] do, or how date object calculations are processed behind the scene.



Main files and what they do:

models.py: defines 5 SQLAlchemy models User, Song, Playlist, PlaylistSong, and Notification.

app.py: Setup of the app. Gets the mixtape.db as the default data, disables Track Modifications to save memory and registers each endpoint as it separate url to keep everything organized.

/routes: Organize the URLs or routes and calls the corresponding services necessary for each action. Also calls the necessary data to do each action.

/services: contains functions that process input arguments and manipulate data. These functions package the results into structured formats—like lists or dictionaries—to cleanly pass the data along to the next part of the app.


Data flow: POST songs/<song_id>/listen in routes/songs.py. The listen() function calls the record_listening_event() function in the /services/streak_service.py file which calls the update_listening_streak() in the /services/streak_service.py file and update a user's listening streak based on their last listening date and their last time listening. This does not return anything, the streak stays in the user model.





Issue #1 - My listening streak keeps resetting:

How I reproduced it:
Input:
curl -X POST http://127.0.0.1:5000/songs/807790b0-f85f-42cc-8107-d2fb555d4d98/listen -H "Content-Type: application/json" -d "{\"user_id\": \"ff7a87c7-5dcb-45bd-aef2-cf979547f2d0\"}"
Output:
{"id":"5872fe30-b5f6-4f70-bc1b-0af3cf380829","listened_at":"2026-07-11T12:00:00","song_id":"807790b0-f85f-42cc-8107-d2fb555d4d98","user_id":"ff7a87c7-5dcb-45bd-aef2-cf979547f2d0"}
Input:
curl http://127.0.0.1:5000/users/ff7a87c7-5dcb-45bd-aef2-cf979547f2d0/streak
Output:
{"streak":2,"user_id":"ff7a87c7-5dcb-45bd-aef2-cf979547f2d0"}

The behavior was triggered when the now date was Sunday.

How you found the root cause:

I found the root cause by checking the files that have to do with updating the streak: /routs/songs.py, /services/streak_services.py and inside this last file I checked the functions record_listening_event() and update_listening_streak(). When I found the update_listening_streak() I realized I had the spot of the problem.

The root cause:

The condition in update_listening_streak() to add a day to the streak specified that the amount of days since the last listen should be 1 day (correct) AND that the current day should NOT be Sunday (elif days_since_last == 1 and today.weekday() != 6:). This provoked a false in the condition when today's day is a Sunday and the logic would go down to resetting the streak (else: user.listening_streak = 1).

Your fix and side-effect check:

I deleted the "and today.weekday() != 6" to ensure that every day counts as a valid day to add a streak.




Issue #5 - The last song in a playlist never shows up

How I reproduced it:
Input:
curl http://127.0.0.1:5000/playlists/beda05a6-a6d0-4aef-9fea-ed46f9f365cd/songs
Output:
{"count":6,"songs":[{"album":null,"artist":"Street Collective","genre":"hip-hop","id":"11422147-7cd9-4206-888a-75982d18621e","share_note":null,"shared_at":"2026-07-04T22:46:05.708663","shared_by":"e7ae2256-d4a4-436e-9a10-f26005b5344a","tags":["hip-hop"],"title":"Block Party"},{"album":null,"artist":"Nova Blix","genre":"lo-fi","id":"6b4c42c2-6b7c-4ef3-85f1-e849939aa468","share_note":null,"shared_at":"2026-07-04T22:46:05.708663","shared_by":"e7ae2256-d4a4-436e-9a10-f26005b5344a","tags":["lo-fi"],"title":"Late Night Session"},{"album":null,"artist":"Solange K","genre":"r&b","id":"36a6f61f-f8ac-4fba-b859-ee47ff13779a","share_note":null,"shared_at":"2026-07-04T22:46:05.708663","shared_by":"e7ae2256-d4a4-436e-9a10-f26005b5344a","tags":["r&b"],"title":"Golden Hour"},{"album":null,"artist":"Hoop Dreams","genre":"rap","id":"03750302-707c-492e-a87a-3751daeef939","share_note":null,"shared_at":"2026-07-04T22:46:05.708663","shared_by":"e7ae2256-d4a4-436e-9a10-f26005b5344a","tags":["rap"],"title":"Free Throws"},{"album":null,"artist":"Elara Moon","genre":"indie","id":"36930ead-f9d4-43d8-b3d1-e8b1590adb35","share_note":null,"shared_at":"2026-07-04T22:46:05.708663","shared_by":"e7ae2256-d4a4-436e-9a10-f26005b5344a","tags":["indie"],"title":"Soft Landing"},{"album":null,"artist":"Borough Kings","genre":"rap","id":"c1f27486-b930-488f-ad9e-77f9e5e7de43","share_note":null,"shared_at":"2026-07-06T22:46:05.708663","shared_by":"a2c822cc-bc0d-4bfc-bbc0-1663360786cc","tags":["rap","boom bap","hip-hop"],"title":"Crown Heights Anthem"}]}

The behavior was triggered when retrieving the songs in a playlist

How you found the root cause:

I found the root cause by checking the files that have to do with opening a playlist: /routes/playlists.py, /services/playlist_service.py and inside this last file I checked the function get_playlist_songs(). When I found the get_playlist_songs() I realized I had the spot of the problem.

The root cause:

The return statement of the function get_playlist_songs() was returning "[song.to_dict() for song in songs[:-1]]" which explicitly cuts the last element from the list.

Your fix and side-effect check:

I deleted the "[:-1]" to ensure that all the songs from a playlist were iterated through.


Issue #4 — I got notified when a friend added my song to a playlist but not when they rated it

How I reproduced it:
Input:
curl -X POST http://127.0.0.1:5000/songs/807790b0-f85f-42cc-8107-d2fb555d4d98/rate -H "Content-Type: application/json" -d "{\"user_id\":\"ff7a87c7-5dcb-45bd-aef2-cf979547f2d0\", \"score\":\"5\"}"
Output:
{"id":"ad20cc30-bdf4-4d2a-882a-69f23c759a3a","rated_at":"2026-07-08T01:39:38.695486","score":5,"song_id":"807790b0-f85f-42cc-8107-d2fb555d4d98","user_id":"ff7a87c7-5dcb-45bd-aef2-cf979547f2d0"}

Input:
http://127.0.0.1:5000/users/7e0727e5-805b-4f21-b207-441fc341da17/notifications
Output:
{"count":0,"notifications":[]}

The behavior was triggered when someone rated a song and the person that shared the song checks their notifications and they are not being notified.

How you found the root cause:

I found the root cause by checking the files that have to do with sending notifications: /routes/songs.py, /services/notification_service.py and inside this last file I checked the function rate_song(). When I found the rate_song() I realized I had the spot of the problem.

The root cause:

After rating, there was nowhere in rate_song() that created a notification.

Your fix and side-effect check:

I created a create_notification() function inside rate_song() that sends a notification to the song sharer (if they are not the ones rating the song) that displays information regarding the rater, the name of the song, and the score.