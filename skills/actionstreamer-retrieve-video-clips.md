---
name: actionstreamer-retrieve-video-clips
description: Find and retrieve recorded video clips from ActionStreamer devices by device, time range or tag — the evidence-retrieval flow after an incident or a shift.
api: ActionStreamer Web API
generated: '2026-09-06'
method: generated
source: openapi/_original/actionstreamer-openapi-original.json
base_url: https://api.actionstreamer.com/v1
operations:
- POST /v1/videoclip/list
- POST /v1/videoclip/devices/list
- POST /v1/videoclip/devices/tag/list
- GET /v1/videoclip/{videoClipID}
- POST /v1/videoclip/extract/list
- POST /v1/videoclip/concatenate
- POST /v1/device/{deviceID}/tag/list
consequence: mostly read; concatenate enqueues work
---

# Retrieve video clips

Read-first. One step (`concatenate`) enqueues server-side work — it is called out explicitly below.

## Before you start

HMAC-SHA256 sign every request; see `authentication/actionstreamer-authentication.yml`. Note the
platform's list convention: most searches are issued as **POST** to a `/list` path with a JSON filter
body, not as GET with query parameters. This trips up integrators expecting a conventional REST read.

## Steps

1. **Search clips for one device.** `POST /v1/videoclip/list` with a filter body returns the filtered
   clip list. `GET /v1/videoclip/list` returns the unfiltered list.

2. **Search across several devices.** `POST /v1/videoclip/devices/list` takes a device set and
   returns clips for all of them — the right call when you are reconstructing an incident that
   several people were wearing cameras for.

3. **Search by tag and time.** `POST /v1/videoclip/devices/tag/list` returns clips, their clip tags,
   and the unique tag list for a start and end epoch. This is the strongest retrieval primitive in
   the API: it gives you the matching clips and the tag vocabulary in one response.
   `POST /v1/device/{deviceID}/tag/list` lists the tags for one device over an epoch range.

4. **Fetch one clip.** `GET /v1/videoclip/{videoClipID}` returns the clip record. Clip payloads live
   behind the `File` resource — a clip carries a `fileID`, so follow that to the file record for the
   media itself.

5. **List extracted clips.** `POST /v1/videoclip/extract/list` returns clips produced by extraction
   rather than direct capture.

6. **Stitch a range together — this one writes.** `POST /v1/videoclip/concatenate` enqueues a
   concatenate event across a start and end time. It is asynchronous: it queues work rather than
   returning media. There is **no idempotency mechanism** on this call, so a retry after a timeout
   will enqueue the job a second time. Poll rather than retry blindly: check the resulting event with
   `GET /v1/event/{eventID}`.

## Pagination

Undocumented. No page, limit, offset or cursor parameter is described in any guide, and no response
envelope with a total or next-page field is published. You cannot tell from the response whether a
list is complete. Constrain results with the time range in your filter body rather than relying on
paging, and treat a suspiciously round result count as a possible truncation.

## Handling failures

Statuses 200, 400, 401, 419, 500. Errors may be JSON or plain text. Steps 1–5 are reads and safe to
retry. Step 6 is not.

## Related

- `data-model/actionstreamer-data-model.yml` — VideoClip belongs_to Device via `deviceID` and to File via `fileID`.
- `conventions/actionstreamer-conventions.yml`
