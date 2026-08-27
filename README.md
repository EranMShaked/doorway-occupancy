# doorway-occupancy

Estimating how many people are home from a single front-door camera, using
Frigate for detection and n8n for the counting logic. Alerts go to Gotify.

Frigate will happily tell you a person appeared at the door. It won't tell you
whether they came in or went out, and that's the part that actually matters if
you want a headcount. This works out direction from the person's path across
the frame and keeps a running total.

Worth being upfront: the number drifts. A door camera cannot see inside the
house, so the count is inferred entirely from crossings. Two people entering on
one door-opening register as one. Someone who stands at the door long enough
for the track to end there looks identical to someone walking in. If you need a
count you can rely on, put a contact sensor on the door and use that as the
entry signal instead. This is the best you can do with the camera alone.

## How it works

The camera is restreamed once through go2rtc so detection and recording share a
single RTSP connection. Frigate runs person detection on motion only.

The frame is split into two zones: `doorway` (the door itself) and `approach`
(the hallway leading up to it). When Frigate finishes tracking a person it
writes an event plus a timeline of positions.

n8n polls the events API every 10 seconds. For each new completed event it
pulls the timeline, converts each bounding box to its bottom-centre point,
tests that point against the zone polygons, collapses consecutive repeats, and
sums the transitions:

    approach -> doorway   = +1
    doorway  -> approach   = -1

The sum is that event's occupancy delta. Someone walking in scores +1. Someone
leaving scores -1. A delivery driver who walks up, drops a parcel and leaves
traces `approach -> doorway -> approach` and correctly nets to zero. That still
holds if Frigate splits the visit into two separate events, since the deltas
add up either way.

Zone polygons are read from Frigate's config at runtime rather than hardcoded,
so retuning the geometry doesn't quietly break the classification.

### What I tried first, and why it didn't work

Two earlier versions both looked right and were only disproved by walking past
the camera and checking the result.

Reading `event.zones` fails because that field records each zone's *first*
entry and never repeats. A round trip is not representable in it at all, so
walking out and back reads as a plain exit.

Reading only the `entered_zone` timeline entries fails because those fire on a
*transition*. A track that begins already inside a zone never logs one, so a
single-direction crossing produces one data point and can't be compared against
anything.

Using every timeline entry, not just the zone-crossing ones, fixes both.

## Setup

Bring up Frigate:

    cp frigate/config.yml frigate/config/config.yml
    # fill in the camera credentials in the go2rtc block
    docker compose -f frigate/docker-compose.yml up -d

Bring up Gotify, change the admin password, then create an application to get
a push token:

    docker compose -f gotify/docker-compose.yml up -d

Import `n8n/frontdoor-occupancy.json`. Before it will run you need to:

- replace `FRIGATE_LAN_HOST` with a host address your phone can reach, so
  snapshot images in the notifications resolve
- create a Query Auth credential named `Gotify Token` with parameter name
  `token` and your Gotify application token, and attach it to the push node

If n8n runs in a container, host services are reachable at the bridge gateway
`172.17.0.1`, not `127.0.0.1`. The workflow ships with that address already.

## Using it

    curl 'http://localhost:5678/webhook/occupancy'
    curl 'http://localhost:5678/webhook/occupancy/reset?count=3'
    curl 'http://localhost:5678/webhook/occupancy/reset?count=3&stats=1'

The reset endpoint exists because you will need it. `stats=1` also clears the
counters that feed the confidence figure.

Occupancy starts wherever you set it. There is no way to seed it from history.

## Things that cost me time

`model:` has to be a top-level key in Frigate 0.17. Nesting it under the
detector is accepted without complaint and then ignored, so the detector keeps
feeding 320x320 frames into a 300x300 model, crashes on every inference, and
gets restarted by the watchdog about once a minute. The only visible symptom is
`detection_fps` sitting at zero, which looks exactly like an idle camera.

`record.retain` was removed in 0.17. An unknown key doesn't fail loudly, it
drops Frigate into safe mode, where the API returns an empty camera list and
everything else appears normal.

Frigate normalises and rewrites `config.yml` on startup, so what's on disk may
not match what you wrote. Check the live `/api/config` instead.

The camera's burned-in timestamp changes every second and reads as motion, so
detection runs on every frame around the clock until you mask it.

Updating a workflow through the n8n API doesn't swap the running instance
straight away. When the swap lands it overwrites the workflow's static data
with whatever was in the payload, so any state change made in that window is
silently reverted. Deploy, wait, then touch state.

## Limits

The count drifts upward over time, because a person lingering at the door is
indistinguishable from one walking through it. Someone who only ever appears in
the `doorway` zone and never in `approach` produces no transition at all and is
missed entirely.

Neither is fixable with better zone geometry. The information isn't in the
video.
