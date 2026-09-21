# Known bugs

Bugs that are understood but deliberately not fixed yet. Each entry keeps the
full context so the investigation does not have to be redone.

## Widget shows the previous song's title, artist and artwork while the progress bar tracks the new song

**Status:** Open, not being fixed for now (decided 2026-09-21).
**Reproducible:** Not reliably. Trigger unknown. Do not change the song, pause,
seek or send any command to Spotify while the bug is live, it may not come back.

### Symptom

Spotify desktop is playing song B. The widget shows song A's title, artist and
album art, but the progress bar moves correctly for song B.

### Where the wrong data comes from

It is not our code. Windows' own media card (Win+A) shows the same stale song A
with a moving progress bar. Both the widget and Windows read the same data from
SMTC (System Media Transport Controls), so Spotify has stopped telling Windows
about the track change.

Why text and art go stale together but progress does not: in
`media_controller.py`, `_get_media_info_async()` makes two separate calls.

- `session.try_get_media_properties_async()` returns title, artist and the
  thumbnail. Spotify updates this through a "media properties changed" event.
- `session.get_timeline_properties()` returns position and duration. Spotify
  updates this through a separate "timeline changed" event.

Spotify fired the timeline update but never fired the media properties update,
so Windows still holds song A's metadata.

### What was ruled out on our side

- `widget.py` `_apply_media_info()` swaps text and art whenever the title or
  artist differs from what is displayed. It has no cache that could pin the old
  song while the progress bar keeps moving.
- The Web API fallback mode (`_using_api_fallback`) was not involved; the
  stale data is visible in Windows itself.

### Possible triggers to check next time it happens

Check these passively, without changing playback:

- Is Crossfade or Automix enabled in Spotify settings? Track changes mid-stream
  are the most common trigger for Spotify skipping the metadata event.
- Was the track changed from another device (Spotify Connect from a phone)?
- Is it the Microsoft Store Spotify or the spotify.com download? They behave
  differently with SMTC; switching is a legitimate fix if one is buggy.
- Does pause then resume clear it? If yes, Spotify just dropped one event.

### Fix options considered

1. **Fix at the source (preferred).** Find the trigger above and change the
   Spotify setting or install. No code change. Only route that also fixes the
   artwork without the Web API.
2. **Cross-check with Spotify's window title.** Spotify's main window is titled
   `Artist - Song` while playing, readable via Win32 `EnumWindows` and
   `GetWindowText`. Compare with SMTC's title and prefer the window title when
   they disagree. Fixes text only. There is no local source for the artwork, so
   art would stay stale. Standard technique, not a hack, but only half a fix.
3. **Cross-check with the Web API `currently_playing`.** Would fix everything
   including art. Rejected by the user for this bug; they do not want the Web
   API involved here.

A stale-metadata detector is cheap if option 2 is ever done: if the timeline
duration changes, or the position jumps backwards, while title and artist stay
identical, the track almost certainly changed. This belongs in the event-driven
rewrite (`PLAN_event_architecture.md`) rather than as another if-else patch in
`_update_media_info()`.
