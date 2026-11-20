# Blue Pearl kiosk

The guest welcome screen shown on the TV at The Blue Pearl Retreat (Destin, FL).
A Raspberry Pi runs Chromium in kiosk mode on this GitHub Pages site:
https://bluepearlretreat.github.io/blue-pearl-kiosk/

## Files

| File | Purpose |
|---|---|
| `index.html` | The kiosk page the TV shows |
| `control-panel.html` | Edits `guest-config.json` (guest name, checkout date, wake) |
| `guest-config.json` | Current guest settings read by the page |
| `ocean-loop.mp4` | Backup background video, used when the live cam can't play |
| `cam.html` | Older cam-only test page. **Not used by the TV**, and still has the old `watch.hdrelay.io` cam address (see below). |
| icons | Favicons / home-screen icons |

The Pi also keeps a local copy of `index.html` in `~/kiosk/` that it shows only when this site is
unreachable. After changing `index.html` here, re-sync that copy on the Pi.

## Live webcam (note, 2026-11-19)

The background is the Pompano Joe's (Miramar Beach) beach cam hosted by HDRelay:
cam ID `cam_five_star___pompano_joe_s_1781797463071`.

**What happened:** in November 2026, HDRelay moved this cam's stream from `watch.hdrelay.io` to
`east2.hdrelay.io`. The cam ID didn't change, but the old address started returning
`path '...' is not configured`. The TV quietly fell back to `ocean-loop.mp4`.

**What changed:** `index.html` no longer hard-codes the server. Every time it connects, or retries
(every 60 s after a failure, or after the picture freezes for 90 s), `findCamUrl()`:

1. Asks HDRelay's public player API,
   `https://watch.hdrelay.io/api/player-profile/<cam>/p_mqjq0ruimd7h`, for the current `streamUrl`.
   Browsers currently reject that response because HDRelay sends the
   `access-control-allow-origin` header twice. If HDRelay fixes that, this step starts working by itself.
2. Otherwise tries the cam on the last good server, then each name in `CAM_SERVERS`, and uses the
   first one that returns a live playlist (`#EXTM3U`).

If nothing is live, the page plays `ocean-loop.mp4` and keeps retrying.

**If the cam moves again** to a server not in `CAM_SERVERS`:

1. Find the new server name. This works from a terminal, not a browser:
   ```sh
   curl -s https://watch.hdrelay.io/api/player-profile/cam_five_star___pompano_joe_s_1781797463071/p_mqjq0ruimd7h \
     | python3 -c 'import json,sys;print(json.load(sys.stdin)["streamUrl"])'
   ```
2. Add it to `CAM_SERVERS` in `index.html` and commit to `main`. Pages updates in about a minute.
3. Restart the kiosk browser on the Pi through Raspberry Pi Connect: kill the main Chromium process,
   and `kiosk.service` relaunches it.
4. Re-sync the Pi's local `~/kiosk/index.html`.

**Quick check that the cam is live:**
```sh
curl -s https://east2.hdrelay.io/live/cam_five_star___pompano_joe_s_1781797463071/index.m3u8 | head -1
# "#EXTM3U" = live; a JSON error = not on that server
```

A separate scheduled check (outside this repo) watches the cam every 30 minutes and sends an alert
if it goes offline or moves to a server not in `CAM_SERVERS`.
