# Workflow notes

14 nodes, three entry points: a 10-second poll, and two webhooks for reading
and correcting the count.

    Every 10 Seconds -> Build Poll Query -> Fetch Frigate Events
                     -> Prepare New Events -> Fetch Timeline
                     -> Classify, Dedupe & Count -> Push to Gotify

    Every 10 Seconds -> Fetch Zone Config

    GET /occupancy       -> Read Occupancy State -> respond
    GET /occupancy/reset -> Apply Manual Reset   -> respond

## State

Everything lives in `$getWorkflowStaticData('global')`:

- `lastSeenTs` — poll cursor, with a 30 second overlap so events still open on
  the previous poll get picked up once they complete
- `processedIds` — capped at 500, trimmed oldest first
- `occupancyState` — the count and its counters
- `zonePolygons` — cached zone geometry, refreshed each poll

Only events with `end_time` set are counted. Frigate is still appending to an
open event, so classifying early gives a truncated path.

## Timeline fetch

The Fetch Timeline node sets `fullResponse` deliberately. Without it n8n splits
each event's timeline array into one item per entry and assigns sequential
`pairedItem` indices, which attaches one event's timeline to a different event
as soon as two people are detected in the same poll. The classify node resolves
its source event through `pairedItem.item` rather than array position for the
same reason.

## Coordinates

Frigate returns normalised zone coordinates here, but the parser checks
magnitude and divides by the detect resolution if it sees pixel values, so it
survives a config expressed either way.
