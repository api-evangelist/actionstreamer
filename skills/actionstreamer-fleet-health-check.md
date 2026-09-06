---
name: actionstreamer-fleet-health-check
description: Check the health of an ActionStreamer device fleet — battery, temperature, connectivity and last activity — and identify devices that need attention before a shift or deployment.
api: ActionStreamer Web API
generated: '2026-09-06'
method: generated
source: openapi/_original/actionstreamer-openapi-original.json
base_url: https://api.actionstreamer.com/v1
operations:
- GET /v1/device/list/all
- GET /v1/device/list/health
- GET /v1/devicehealth/getlatest/{deviceID}
- POST /v1/devicehealth/list
- POST /v1/device/{deviceID}/health/linechart
- GET /v1/device/{deviceID}
consequence: read-only
---

# Fleet health check

Read-only. Every operation in this skill is a GET or a filter-POST that returns data; nothing here
changes device state.

## Before you start

Every request must be HMAC-SHA256 signed. Send `Authorization: HMAC-SHA256 {access_key}`,
`X-AccessKey`, `X-Signature`, `X-Timestamp` (Unix epoch seconds), `X-Nonce` (a fresh UUIDv4 per
request) and `Content-Type: application/json`. Build the string to sign from METHOD, PATH,
HEADER_STRING, PARAMETER_STRING and BODY joined by newlines and trimmed, removing Content-Type from
the header set first, sorting header keys and parameters, and normalizing the path. Full rules:
`authentication/actionstreamer-authentication.yml`.

The ActionStreamer OpenAPI declares no `operationId` values, so operations below are named by method
and path exactly as the provider publishes them.

## Steps

1. **Get the fleet.** `GET /v1/device/list/all` returns every device visible to the authenticated
   account. Use `GET /v1/device/list` for the narrower default list. If you were given a name or
   serial number rather than an ID, resolve it first with `POST /v1/device/name`.

2. **Pull health for the whole fleet in one call.** `GET /v1/device/list/health` returns device
   health rows for the authenticated user's devices. Prefer this over looping per device — the
   API publishes no rate limits, which means no published headroom either, so minimise call volume.

3. **Drill into a specific device.** `GET /v1/devicehealth/getlatest/{deviceID}` returns the most
   recent health record for one device. `POST /v1/devicehealth/list` takes a filter body when you
   need a range of records rather than the latest.

4. **Chart a variable over time.** `POST /v1/device/{deviceID}/health/linechart` returns line-chart
   series for a single device health variable — use it to distinguish a device that is briefly cold
   from one that has been degrading across a shift.

5. **Correlate with the device record.** `GET /v1/device/{deviceID}` gives you the device's
   configuration and identity alongside the telemetry.

## What the health record covers

The platform's device-health surface (mirrored by the `DeviceHealthFunctions` module in the
first-party Python library) reports battery percentage and voltage, charge status, CPU and GPU
temperature, CPU and memory and disk usage, throttling status, Wi-Fi link quality and radio info,
IP addressing, camera activity, microphone and audio-output levels, system uptime and system time.

## Handling failures

Documented statuses are 200, 400, 401, 419 and 500. Treat any non-2xx as a failure **even when the
body is a plain string** — the provider states the error envelope is not standardized and clients
must handle both JSON and plain text. On a 401, re-check the canonical string before assuming the
key is bad; log the string-to-sign and headers while validating auth. There is no documented 429 and
no `RateLimit-*` header, so back off on your own schedule.

Retries are safe here because every operation in this skill is a read. Do not carry that assumption
into the write skills — the API publishes no idempotency mechanism.

## Related

- `conventions/actionstreamer-conventions.yml` — pagination is undocumented; do not assume a list response is complete.
- `errors/actionstreamer-problem-types.yml`
- Python alternative: `pip install actionstreamer`, then the `WebService.Health` and `DeviceHealthFunctions` modules.
