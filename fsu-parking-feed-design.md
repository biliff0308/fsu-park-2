# FSU Parking Feed Design — Proposal

A clean replacement for `count.json` and `count2.html`, with APDS-aligned vocabulary
where it fits, scoped to what FSU actually needs.

## Design principles

1. **Two feeds, two cadences.** Static facility data (rarely changes) and dynamic
   availability data (every 5 minutes) are separate. The client caches the static
   data aggressively and polls only the dynamic data on its refresh cycle.
2. **No presentation logic in the data layer.** No gauge cell counts. No color codes.
   The client decides visualization based on numbers.
3. **Borrow APDS vocabulary where it fits.** Field names, enums, and structure mirror
   APDS where the concepts align. This makes the feed legible to anyone who knows
   the standard and gives FSU CS/MIS students exposure to real industry conventions.
4. **Reject APDS complexity where it doesn't fit.** No RightSpecifications, no
   versioning, no MultilingualString, no SubPlace hierarchy. Six garages don't
   need a generalized hierarchy element model.
5. **Explicit data quality signals.** Replace magic status codes (`-2` for bad data)
   with a clear `dataQuality` enum.
6. **Combined student + faculty data per garage.** The LPR feed and the sensor feed
   are merged server-side. The client sees one unified availability number per
   permit type per garage.

## Two endpoints

### `GET /api/v1/places`

Static facility data. Updates rarely (only when garages are added, renamed, or have
their capacity changed). Client fetches on app launch and caches; refetches on a
long interval (daily or on demand).

### `GET /api/v1/availability`

Dynamic availability data. Updates every 5 minutes server-side. Client polls every
5 minutes while in foreground.

## Schemas

### Static feed: `GET /api/v1/places`

```json
{
  "campus": {
    "id": "fsu-tallahassee",
    "name": "Florida State University",
    "timeZone": "America/New_York"
  },
  "places": [
    {
      "id": "2100",
      "name": "Traditions Way Garage",
      "type": "garage",
      "location": {
        "type": "Point",
        "coordinates": [-84.297441, 30.441505]
      },
      "permitTypes": ["students", "staff"],
      "capacities": [
        { "permitType": "students", "totalSpaces": 672 },
        { "permitType": "staff", "totalSpaces": 0 }
      ]
    },
    {
      "id": "2020",
      "name": "Spirit Way Garage",
      "type": "garage",
      "location": {
        "type": "Point",
        "coordinates": [-84.305791, 30.442898]
      },
      "permitTypes": ["students", "staff"],
      "capacities": [
        { "permitType": "students", "totalSpaces": 1243 },
        { "permitType": "staff", "totalSpaces": 123 }
      ]
    }
  ]
}
```

Notes on the static feed:

- `id` is the existing `facuid` from the legacy feed, as a string. Stable across renames.
- `location` is GeoJSON Point — `[longitude, latitude]` (note the order, GeoJSON convention).
  Decimal degrees, not your integer encoding.
- `type` is currently always `"garage"` for FSU. Reserved for future (`"lot"`, `"street"`).
- `permitTypes` uses APDS `UserTypeEnum` values — `"students"` and `"staff"` are the FSU
  W/R distinction expressed in standard vocabulary.
- `capacities` is an array because a garage can host multiple permit types with
  different totals. If `totalSpaces` is 0, that permit type isn't sold at this garage.

### Dynamic feed: `GET /api/v1/availability`

```json
{
  "generatedAt": "2026-05-13T11:20:00-04:00",
  "places": [
    {
      "id": "2100",
      "availability": [
        {
          "permitType": "students",
          "occupied": 300,
          "available": 372,
          "lastUpdated": "2026-05-13T11:19:34-04:00",
          "source": "spaceSensor",
          "dataQuality": "ok"
        },
        {
          "permitType": "staff",
          "occupied": 0,
          "available": 0,
          "lastUpdated": "2026-05-13T11:19:34-04:00",
          "source": "anpr",
          "dataQuality": "unavailable"
        }
      ]
    },
    {
      "id": "2165",
      "availability": [
        {
          "permitType": "students",
          "occupied": null,
          "available": null,
          "lastUpdated": "2026-05-13T11:19:47-04:00",
          "source": "spaceSensor",
          "dataQuality": "sensorFault"
        }
      ]
    }
  ]
}
```

Notes on the dynamic feed:

- `generatedAt` is when the feed was assembled server-side. ISO 8601 with timezone.
- Each place has an `availability` array keyed by permit type. Lets students and
  staff have independent availability numbers in the same garage.
- `occupied` and `available` are explicit. Either can be `null` when `dataQuality`
  indicates the number isn't trustworthy.
- `lastUpdated` is per-record because the sensor and the LPR system update at
  different times. Client can show staleness per permit type.
- `source` uses APDS `ParkingSpaceOccupancyDetectionEnum` values — `spaceSensor`,
  `anpr`, etc. Tells the client (and a savvy user) how the number was determined.
- `dataQuality` replaces the legacy magic status codes. Enum: `ok`, `stale`,
  `sensorFault`, `unavailable`.

## Enums

### `permitType` (subset of APDS UserTypeEnum)
- `students` — white (W) spaces in legacy terms
- `staff` — red (R) spaces in legacy terms
- (Future: `visitors`, `disabled`, etc. if needed)

### `source` (subset of APDS ParkingSpaceOccupancyDetectionEnum)
- `spaceSensor` — physical sensor in each space
- `anpr` — license plate recognition (the LPR feed)
- `visual` — manual count

### `dataQuality` (new, FSU-specific)
- `ok` — data is current and trustworthy
- `stale` — last update is older than expected; counts may be drifting
- `sensorFault` — sensor is reporting impossible values (the negative-occupied case)
- `unavailable` — no data source is reporting for this permit type at this place

### `type` (place type, FSU-specific subset of APDS ElementDescriptorEnum)
- `garage` — multi-level parking structure
- `lot` — surface lot
- `street` — on-street parking zone

## Announcements feed (separate from this proposal)

The `_announce.json` feed is a separate concern. It should also be modernized
(add timestamps, fix UTF-8 encoding, give announcements a created/expires field),
but it's not tied to the parking-data redesign and can be tackled separately.

## What this design improves over the legacy feeds

| Concern | Legacy | New |
|---|---|---|
| Coordinates | Integer-encoded `"30441505"` | GeoJSON Point, decimal degrees |
| Timestamps | US format `"5/13/2026 11:19:34 AM"`, no TZ | ISO 8601 with TZ offset |
| Bad-data signal | `status: -2`, `occupied: -169` | `dataQuality: "sensorFault"`, nulls |
| Permit types | `factype: "W"` / `"R"` (FSU-only jargon) | `permitType: "students"` / `"staff"` (standard) |
| Capacity | Implicit / unavailable | Explicit per permit type |
| Detection method | Implicit (assumed sensor) | Explicit `source` field |
| Stale records | Mixed with active records | Static feed has only active places |
| Presentation logic | `status` = gauge cell count | None; client decides |
| Encoding | Latin-1 corruption (`â` for em-dash) | UTF-8 throughout |
| Field names | Misspelled (`avaliable`) | Correct |

## What's deliberately NOT in this design

- **Real-time push.** The dynamic feed is poll-based. Push (WebSockets, SSE) is
  overkill for a 5-minute update cadence.
- **Per-space data.** APDS supports this via the `Space` hierarchy element. FSU
  doesn't track individual spaces; we report at the garage level only.
- **Historical data / time series.** Each call returns current state. If FSU later
  wants "occupancy over time" for the predictive analytics idea, that's a different
  feed.
- **Pricing, hours, payment methods.** All APDS supports, none FSU needs in v2.
- **Pagination.** Six garages don't need it. Add only if the place count grows
  substantially.
- **Authentication.** Public read-only feed. If FSU later wants to restrict access,
  add an API key header.

## Versioning

URL versioning: `/api/v1/places` and `/api/v1/availability`. When the schema
needs to break backward compatibility (which shouldn't happen often), increment
to `/api/v2/`. Legacy clients (including the current legacy app and external
consumers) keep working.

The legacy `count.json` and `count2.html` endpoints continue to serve as-is for
now. Eventually they can be deprecated, but not on a tight schedule.

## Server-side implementation note

This is the contract. The server implementation (what code generates these JSON
responses) is a separate concern. Likely a small service that:

1. Polls the existing sensor data source for student-space counts.
2. Polls the LPR system for staff-space counts.
3. Joins them with the static garage catalog.
4. Renders the unified JSON.

The exact stack (Python/Flask, ASP.NET, whatever you prefer) is up to you. The
feed contract is independent of how it's built.
