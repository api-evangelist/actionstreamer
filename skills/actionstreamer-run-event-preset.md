---
name: actionstreamer-run-event-preset
description: Trigger a camera action on an ActionStreamer device by running a configured event preset, then track it to completion or stop it — the platform's primary write flow.
api: ActionStreamer Web API
generated: '2026-09-06'
method: generated
source: openapi/_original/actionstreamer-openapi-original.json
base_url: https://api.actionstreamer.com/v1
operations:
- GET /v1/eventpreset/list/device/{deviceID}
- GET /v1/eventpreset/{eventPresetID}
- POST /v1/eventpreset/run/{eventPresetID}
- GET /v1/event/{eventID}
- POST /v1/event/list/pending
- DELETE /v1/event/list/pending/{deviceID}
- POST /v1/event/endprocess/{eventID}
- GET /v1/device/{deviceID}/stream/mostrecent
consequence: write — actuates physical hardware
---

# Run an event preset

**This skill writes, and the write reaches a physical device worn by a person.** Running a preset
queues a real command to a camera in the field. Read the safety section before automating it.

## Safety rules for an agent

- **There is no idempotency mechanism.** The API publishes no `Idempotency-Key` header and nothing
  in the contract matches "idempoten". If `POST /v1/eventpreset/run/{eventPresetID}` times out, you
  do **not** know whether it fired. Do not blindly retry — verify with step 4 first.
- **The cancel window is narrow and undocumented.** A queued event can be cleared only while it is
  still pending; once the device agent dequeues it, `DELETE /v1/event/list/pending/{deviceID}` no
  longer helps and you must use `POST /v1/event/endprocess/{eventID}` instead. ActionStreamer does
  not publish this boundary — it is inferred from the operation set. Treat the window as "seconds,
  possibly already gone."
- **Confirm with a human before firing** unless you have been given explicit standing authority for
  this device. Ask which device, which preset, and confirm the preset's configured action.

## Steps

1. **List the presets for the device.** `GET /v1/eventpreset/list/device/{deviceID}` returns the
   presets configured for that device. Never guess a preset ID — a preset ID is meaningful only in
   the context of its device.

2. **Read the preset before running it.** `GET /v1/eventpreset/{eventPresetID}` returns its
   configuration. Confirm the action, the agent type and the target device match what was asked for.

3. **Run it.** `POST /v1/eventpreset/run/{eventPresetID}`. This enqueues the event; it does not
   execute synchronously. Capture the returned event identifier — it is the only handle you have on
   the work.

4. **Track it.** `GET /v1/event/{eventID}` returns the event and its status. `POST /v1/event/list/pending`
   returns pending events if you need to see the queue. Poll here rather than retrying step 3.

5. **Stop it if you need to.**
   - Still pending, never picked up: `DELETE /v1/event/list/pending/{deviceID}` clears pending events
     for the device. Note the granularity — this clears *the device's pending queue*, not just your
     event. If other work is queued, you will clear that too.
   - Already running: `POST /v1/event/endprocess/{eventID}` queues an event to end the process.

6. **Pick up the resulting stream, if the preset started one.**
   `GET /v1/device/{deviceID}/stream/mostrecent` returns the most recent stream for the device,
   including its `publishURL` (an `srt://` ingest on media.actionstreamer.com) and `readURL` (a
   WebRTC playback URL). `GET /v1/device/{deviceID}/stream/list` lists them all.

## How the device sees this

The device agent pulls work rather than receiving a push: it calls `POST /v1/event/dequeue` for the
next pending event, or long-polls `POST /v1/event/list/pending/longpoll`. There are no webhooks
anywhere in this API, so there is no callback to subscribe to — polling `GET /v1/event/{eventID}` is
the only completion signal available to you.

## Handling failures

Statuses 200, 400, 401, 419, 500, and the body may be plain text rather than JSON. A 5xx on step 3 is
genuinely ambiguous: go to step 4 and look for the event before doing anything else. If you cannot
determine whether the event was created, say so plainly rather than firing again.

## Related

- `conventions/actionstreamer-conventions.yml` — the reversibility block covers each write surface and its (unpublished) window.
- `authentication/actionstreamer-authentication.yml`
- Provider sample: https://developer.actionstreamer.com/docs/Sample_Code/run_event_preset
