---
name: confirmed-api
description: Read and write a Confirmed production over its HTTP API - shows and their spots, show meta, ticket sales, show types, venues and rooms, acts, the team and file uploads - plus the public shows feed. Use when writing code that talks to confirmed.show, syncs bookings, ticketing, a website or a CRM with Confirmed, or when someone mentions a Confirmed API key or production handle.
---

# Confirmed API

Confirmed (https://confirmed.show) is a booking platform for comedy productions. A **production** is one account: a
comedy club, a promoter, a festival run. Its API reads and writes that production's shows, the spots on them, venues,
acts, ticket sales and team.

Full docs: https://confirmed.show/llms.txt, with every page as markdown at `https://confirmed.show/api-docs/<page>.md`.
Everything you need for ordinary work is below. Read a page only for a field this skill doesn't cover.

## Base URL and authentication

```
https://confirmed.show/api/v1/<production>/<endpoint>
```

- `<production>` is the production's handle, the slug in its Confirmed URLs. Leaving it out gives a `404` that says so.
- Send the key as `Authorization: Bearer <key>`. `X-API-Token: <key>` and `?token=<key>` work too.
- Keys come from the production's **Settings > Data & API** page (https://confirmed.show/to/data). Keep them in an
  environment variable such as `CONFIRMED_API_KEY`, never in code or a browser bundle.
- Every endpoint allows cross-origin calls, but a key in browser code is readable by anyone. Call from a server.

| Key type | Reaches |
| --- | --- |
| Read-only | Every `GET` in the private API |
| Read-write | The same, plus creating and updating shows, acts and venues, sending ticket sales, and file uploads |
| Show control | Only the Showtime endpoints (see the `confirmed-showtime` skill) |

## Conventions

- **JSON in, JSON out.** Send `Content-Type: application/json` on writes.
- **Ids** are short random strings (`nan3`, `b9b4`). Show types use readable ids (`friday-showcase`).
- **Timestamps** come back as `YYYY-MM-DD HH:MM:SS` with no timezone. A show's `ts` is local time at its venue;
  the venue's `tz` says which zone. Ticket `updated_ts` and history are UTC. On input, ISO 8601 such as
  `2026-08-31T20:00` or `2026-08-31 20:00` both work.
- **Partial updates.** `PUT` changes only the fields you send. `null` clears a field.
- **Unknown fields are errors.** A misspelt field comes back as a `422`, never silently ignored.
- **References are checked.** Show types, venues, rooms, acts and team members you name must already exist in the
  production, or you get a `422` naming the one that doesn't.
- **Writes reach the app live.** Open Confirmed tabs update straight away, so there's no need to tell anyone to refresh.
- **Records made through the API** carry `created_by_api_key: {"key": "<first 4 chars>", "description": "..."}`.

### Errors

| Status | Body | Meaning |
| --- | --- | --- |
| `400` | plain text | No API token |
| `401` | plain text | Unknown token |
| `403` | plain text | Wrong key type, usually read-only on a write |
| `404` | `{"error": "..."}` | No record with that id, or unknown endpoint |
| `409` | `{"error": "...", "id": "..."}` | Conflict, such as an act email that another act has (`id` is that act) |
| `422` | `{"error": "..." or [...]}` | Validation. A list is pydantic-style: `[{"loc": [...], "msg": "...", "type": "..."}]` |

Batch writes (`PUT /shows`, `PUT /tickets`) check everything first and write nothing if any part fails.

## Endpoints

| Method | Path | Key | Purpose |
| --- | --- | --- | --- |
| GET | `/shows` | read | Shows in a window: `ts_from`, `ts_to` (default 10 days back to 30 ahead), `show_type` |
| GET | `/shows/{id}` | read | One show |
| POST | `/shows` | write | Create one show per date |
| PUT | `/shows` | write | Update several shows: `{"updates": [{"id", "changes"}]}` |
| PUT | `/shows/{id}` | write | Update one show: body is the changes |
| GET | `/meta-fields` | read | Show meta and show type metadata fields, with names and types |
| GET | `/tickets` | read | Sales per show in a window: `ts_from`, `ts_to`, `show_type` |
| GET | `/shows/{id}/tickets` | read | One show's sales, with `history` |
| PUT | `/shows/{id}/tickets` | write | Set one show's sales from one source |
| PUT | `/tickets` | write | Set sales for many shows: `{"updates": [{"show_id", "source", "sales"}]}` |
| GET | `/show-types` | read | Show types with their metadata; `archived=true` includes archived ones |
| GET | `/venues` | read | Venues with rooms; `name` filters |
| GET | `/venues/{id}` | read | One venue |
| POST | `/venues` | write | Add a venue |
| PUT | `/venues/{id}` | write | Update a venue and its rooms |
| GET | `/acts` | read | Acts; `name` filters, `tags=a,b` needs all of them |
| GET | `/acts/{id}` | read | One act |
| POST | `/acts` | write | Add an act |
| PUT | `/acts/{id}` | write | Update an act |
| GET | `/team` | read | Everyone with access, with role and permissions |
| POST | `/upload-file?filename=x.png` | write | A signed upload slot for a file |

## Shows

A show has a `type` (show type id), `ts`, a venue and room, and spots split into `hosts`, `lineup` (acts and breaks)
and `production` (crew), each in running order.

```json
{
  "id": "nan3", "type": "friday-showcase", "ts": "2026-07-21 21:00:00", "doors_ts": "2026-07-21 20:30:00",
  "venue_id": "5c9g", "room_id": "bpjj", "status": null, "public": true, "tags": [], "description": "",
  "production_contacts": ["ejcs"], "currency": "GBP", "comment": null, "cancelled_ts": null,
  "meta": {"tickets_url": "https://tickets.example.com/nan3"},
  "created_by_api_key": null,
  "hosts": [{"id": "a1b2", "category": "host", "act_id": "ejcs", "role": "MC", "status": "confirmed", "order": 0}],
  "lineup": [{"id": "c3d4", "category": "act", "act_id": "b9b4", "spot_length": 20, "fee": "150", "status": "offered", "order": 1}],
  "production": []
}
```

**Create** with `POST /shows`. One show per entry in `dates`, all sharing the rest:

```json
{
  "dates": ["2026-07-21", "2026-07-28"],
  "time": "21:00",
  "type": "friday-showcase",
  "production_contacts": ["ejcs"],
  "venue_id": "5c9g", "room_id": "bpjj",
  "doors": 30, "public": false, "tags": ["summer"], "description": "",
  "hosts": [{"role": "MC"}],
  "lineup": [{"actID": "b9b4", "spot_length": 20}, {}, {}],
  "production": [{}]
}
```

- Required: `dates`, `time` (24-hour `HH:MM`), `type`, `production_contacts` (at least one, each an `act_id` from
  `/team`).
- With no spot lists at all you get one host, three acts and one production spot. Act spots default to 10 minutes.
- Returns `{"created": 2, "ids": ["nan3", "p4rt"]}`, not the shows. Fetch them if you need them.

**Spot fields:** `actID` (or `act_id`), `category` (`act`, `host`, `production`, `break`, `other`), `order`,
`spot_length` (minutes), `role`, `fee` (a string), `arrive_mins`, `leave_mins`, `call_time`, `spot_start`,
`spot_end`, `mc_opens` (host only), `status` (`planning`, `offered`, `confirmed`, `rejected`, `cancelled`, `no-show`).

**Update** fields: `type`, `ts`, `venue_id`, `room_id`, `doors`, `public`, `tags` (replaces), `description`,
`production_contacts`, `status` (`"cancelled"` cancels, `""` puts it back on), `currency`, `comment`, `meta`, and
`spot_updates`.

- `spot_updates` is `{spot_id: {field: value}}` and patches spots in place. It can't add or remove spots, and a spot
  id that isn't on the show is a `422`.
- A spot on a settlement can't change its act, status, fee, role or length.
- Prefer cancelling (`"status": "cancelled"`) to anything destructive. There are no deletes in the API.

## Show meta and show type metadata

Two different things, both called `meta`:

| | Show meta | Show type metadata |
| --- | --- | --- |
| Lives on | Each show (`show.meta`) | Each show type (`show_type.meta`) |
| API writes it | Yes, through `PUT /shows` | No, read only |
| Built-ins | `tickets_url`, `event_url`, `promo_image`, `promo_thumb`, `social_card`, `promo_video`, `poster`, `promo_blurb` | None |
| Name style | `snake_case` | `camelCase` |
| Groups | No | Yes, arrays of objects |

- `GET /meta-fields` returns `{"show_fields": [...], "show_type_fields": [...]}`. Each field has `name`, `label`,
  `type` and `description`. Show fields add `built_in` and `hidden`, and groups add `fields`. Check it before writing
  a custom field, because names are production-specific.
- Types: `short-text`, `long-text`, `url`, `number`, `boolean`, `image`, `video`, `file`, plus `show-types` and
  `group` on show types only. `url`, `image`, `video` and `file` take a string starting `http://` or `https://`.
- Meta merges field by field. `{"meta": {"tickets_url": "..."}}` leaves the other fields alone, and
  `{"meta": {"tickets_url": null}}` clears one.
- `hidden` only takes a built-in off the show card in the app. The API still reads and writes it.

## Ticket sales

`tickets_sold` on a show is a total plus a breakdown with one row per ticket type per source.

```json
{"source": "eventbrite", "sales": [{"type": "Standard", "price": 15, "sold": 48}, {"type": "Concession", "price": 10, "sold": 12}]}
```

- Always send **running totals**, never sales since the last call. Each call replaces that `source`'s rows on the
  show, so sending the same numbers again is harmless.
- `source` names the feed (letters, digits, underscores, starting with a letter; default `api`). Use one per ticket
  seller, or one seller's numbers will overwrite another's.
- An empty `sales` list removes that source's numbers from the show.
- Rows from Confirmed's own ticket imports appear in the breakdown with their `source_type` and `source: null`. Your
  calls never touch them.

## Venues

- `POST /venues` requires `name`, `tz` (IANA, such as `Europe/London`), `latitude` and `longitude`. Get coordinates
  from a geocoder. `utc_offset_minutes` is optional and worked out from `tz`.
- Other fields: `country` (full name, such as `United Kingdom`), `city`, `address`, `postcode`, `description`,
  `parking`, `public_transport`, `phone_number`.
- `rooms` on create: `[{"name", "description", "capacity", "wheelchair_access", "green_room", "di_box",
  "projector_info"}]`. With no rooms, the venue gets one called "Performance space".
- `PUT /venues/{id}` takes the same fields plus `room_updates` (`{room_id: {field: value}}`) and `new_rooms`. Rooms
  can't be removed through the API.

## Acts

An act is a performer (or crew member) the production books. If the act has their own Confirmed account, the record
merges the production's copy with the act's profile, and `deltas` shows both: `{"prod": {...}, "act": {...}}`.

- Writable fields: `name`, `email`, `phone`, `legal_name`, `notes`, `tags`, `bio`, `bio_extended`, `credits`,
  `plug`, `website`, `instagram`, `twitter`, `tiktok`, `facebook`, `video_link`.
- `POST /acts` needs `name` and returns the act with `201`. An email another act has returns `409` with that act's
  `id`. Use it to update the existing act instead of creating a duplicate.
- On a **linked** act (one with `global_id`), writing a profile field such as `bio` keeps the production's value
  over the act's and adds it to `use_prod`. `null` drops the production's value so the act's own shows again.
  `email` can't change on a linked act (`422`). `notes` and `tags` are always the production's own.
- Act records include contact and payment details. Treat responses as personal data.

## Team

`GET /team` lists everyone with access: `act_id`, `name`, `email`, `images`, `role`, `status` and `permissions`.
`production_contacts` on shows takes these `act_id`s.

## Files

Image, video and file fields store URLs. To upload your own:

1. `POST /upload-file?filename=promo.png` returns `{"upload_url", "file_url"}`.
2. `PUT` the file's bytes to `upload_url` with its `Content-Type`, within fifteen minutes.
3. Save `file_url` where you need it, such as `{"meta": {"promo_image": file_url}}` on a show.

Uploaded files are public to anyone with the URL, and they aren't resized or cropped.

## Public endpoints (no key)

- `GET https://confirmed.show/api/v1/<production>/shows.json` lists public, non-cancelled shows from 30 days back to
  `future_shows_limit` days ahead (default 30). It includes confirmed hosts and acts with bios and socials, built-in
  meta and `tickets_available`, but no fees, contacts or crew. `show_types=a,b` filters, and `id` fetches one show.
- `GET https://st.confirmed.show/api/<production>/show-types.json` is the published show type metadata. It updates
  only when someone clicks Publish in Confirmed, so use the private `/show-types` for live values.

Use these for anything that runs in a browser.

## Example

```python
import os
import requests

BASE = "https://confirmed.show/api/v1/my-production"
session = requests.Session()
session.headers["Authorization"] = f"Bearer {os.environ['CONFIRMED_API_KEY']}"

shows = session.get(f"{BASE}/shows", params={"ts_from": "2026-07-01", "ts_to": "2026-07-31"}).json()
for show in shows:
    response = session.put(f"{BASE}/shows/{show['id']}/tickets", json={
        "source": "eventbrite",
        "sales": [{"type": "Standard", "price": 15, "sold": 48}],
    })
    response.raise_for_status()
```

Questions or something not behaving as described: support@confirmed.show
