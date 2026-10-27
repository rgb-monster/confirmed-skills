---
name: confirmed-showtime
description: Drive a live Confirmed show from outside the app - move the running order on or back, toggle the break, and read, set or clear which show is live - with a show control key. Use when wiring a lighting desk, QLab, a stream deck, a button box or a timer to Confirmed's Showtime, or when someone mentions a Confirmed show control key.
---

# Confirmed Showtime API

Showtime is the part of Confirmed (https://confirmed.show) that runs a show on the night: the running order, the cue
screen for acts and the break. These endpoints let outside kit drive it.

Full docs: https://confirmed.show/api-docs/show-control.md and https://confirmed.show/api-docs/current-show.md

## Authentication

```
https://confirmed.show/api/v1/<production>/<endpoint>
```

- `<production>` is the production's handle.
- These endpoints need a **show control** key from **Settings > Data & API** (https://confirmed.show/to/data). It
  reaches these two endpoints and nothing else, so it's safe to leave on a machine backstage.
- Send it as `Authorization: Bearer <key>`, `X-API-Token: <key>`, or `?token=<key>` for kit that can only fire a URL.

## Which show

Both endpoints act on the **current show**. By default Confirmed works it out from the schedule: a show stays live
until 30 minutes past its scheduled finish, or until five minutes before the next one starts, whichever comes first.
When nothing is running, the next show on the schedule counts as current.

When a production runs several shows at once, narrow it down with query parameters:

| Parameter | Meaning |
| --- | --- |
| `show_id` | This show, skipping the search |
| `venue_id` | Only shows at this venue |
| `room_id` | Only shows in this room. Needs `venue_id` too. |
| `show_type` | Only shows of this type |

## Control the show

`POST /shows/control?action=<action>` (`GET` works too, for kit that can only send `GET`)

| `action` | Effect |
| --- | --- |
| `next` | Move the running order on to the next spot |
| `prev` | Move back one spot |
| `toggle-break` | Start the break, or end it |

```bash
curl -X POST "https://confirmed.show/api/v1/my-production/shows/control?action=next&venue_id=5c9g" \
  -H "Authorization: Bearer $CONFIRMED_CONTROL_KEY"
```

Returns `{"status": "ok", "command_id": "<uuid>"}`. `command_id` is unique per command, which helps when logging what
a desk sent.

## Read, set or clear the current show

`/shows/current`:

| Method | Effect | Response |
| --- | --- | --- |
| `GET` | Which show is marked current | `{"show_id": "ab12"}`, or `{"show_id": null}` |
| `POST` | Mark a show current: the `show_id` you pass, or the one the schedule says is live | `{"status": "ok", "show_id": "ab12"}` |
| `DELETE` | Clear it, so Confirmed goes back to the schedule | `{"status": "ok", "show_id": null}` |

Clear the current show at the end of a run, or last week's show stays current for the team.

## Errors

| Status | Body | Meaning |
| --- | --- | --- |
| `400` | plain text | No key |
| `401` | plain text | Unknown key |
| `403` | plain text | The key isn't a show control key |
| `404` | `{"error": "Show not found"}` | No show has the `show_id` you passed |
| `404` | `{"error": "No current show found"}` | Nothing live matched the parameters |
| `422` | `{"error": "room_id can only be used together with venue_id"}` | `room_id` without `venue_id` |
| `422` | `{"error": [...]}` | `action` missing or not one of the three |

## Wiring tips

- QLab: a Network cue, or a Script cue running `curl`, with the full URL including `?token=` when headers aren't
  available.
- Stream Deck: a "Website" or HTTP request action pointed at the `next` URL.
- Don't fire `next` from a timer without a person watching. Acts overrun, and the running order should follow the room.

Questions: support@confirmed.show
