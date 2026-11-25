# Design: themed-sets

Status: **draft** (awaiting approval)

System-level decisions are in the [decision log](../architecture.md#9-decision-log) (D1 to D7) and are not repeated here.

## 1. Goals and non-goals

Goals

- A **varied single daily notification**: with one window per day, the event type is picked from a list each day instead of always being a memory.
- New event types that show a **set of 3 to 6 photos** (default 4) as one collage:
  - `context`: a theme found with Immich's CLIP search ("sunset", "snow", "beach", ...), spread across years.
  - `person_years`: one named person, spread across years.
  - `album_set`: a highlight from a chosen album or an `[Auto]` event album, instead of a single surprise photo.
- **No new privacy features.** The new event types follow upstream's existing privacy rules, and a test proves they never use locked or archived photos.
- Everything configurable in `config.yaml`; defaults keep upstream behaviour when the new options are absent.
- Small, well tested hooks into upstream files so merging upstream stays easy.

Non-goals

- Matching Google Photos exactly (its rules are not public).
- Replacing upstream scheduling, repeat tracking, screenshot skipping, deep links or the dashboard.
- Training models: only Immich's own faces, CLIP embeddings and metadata are used.
- Multi user changes in the first version; the new options are per user ready.

## 2. Interfaces and data contracts

### 2.1 Configuration (additions under `settings`, all optional)

```yaml
asset_repeat_days: 180            # existing key, value changed (D7)
notification_windows:
  - start: "10:00"
    end: "23:00"
    events: [memory, context, person_years, album_set, person, then_and_now, collage]
    varied: true                  # NEW: do not force a memory; pick from events (weighted)
themed:
  photos_per_set: {min: 3, max: 6, default: 4}
  context_queries: ["sunset", "beach", "snow", "food", "mountains", "birthday party"]
  weights: {memory: 4, context: 2, person_years: 2, album_set: 2, person: 1, then_and_now: 1, collage: 1}
  cooldown_days: {context: 10, person_years: 14, album_set: 7}
```

### 2.2 State

Reuses upstream `state/state.json` (sent photos per user). New keys only add per theme cooldown timestamps (`themed_last`), keyed by theme. Safe to delete.

### 2.3 Immich API

Existing upstream calls plus `POST /api/search/smart` (CLIP). Each is covered by a test against a recorded v3 response.

### 2.4 Notification output

Same shape as upstream's collage notifications: title, one short message line, one attached collage, `app_click_url` deep link to the first asset (or the album).

## 3. Design

### 3.1 Varied selection

In the priority engine, when a window has `varied: true`, skip the "window 1 always memory" rule and pick from `events` by weighted random among the types that are eligible (cooldown passed, enough photos found after privacy and repeat filtering). Birthday still pre-empts. If nothing qualifies, send nothing (a skipped day is better than a weak notification).

### 3.2 Candidate generators (new)

- **context**: pick a query from `context_queries` not on cooldown, call smart search (about 100 results), keep photos that pass the privacy filter, then choose `default` photos spread over distinct years.
- **person_years**: pick a named person not on cooldown, take their photos, round robin by year.
- **album_set**: pick an album from the user's `album_names` or any `[Auto]` album with at least 8 photos, choose a spread.

Each returns the same structure as upstream generators, then goes through upstream's repeat tracking (`asset_repeat_days`) and collage builder.

### 3.3 Privacy

No new rules. The new generators reuse upstream's candidate filtering (hidden people, app-generated images and screenshots are skipped). The only addition is a test (section 5) that proves locked and archived photos can never appear in a `context`, `person_years` or `album_set` notification.

### 3.4 Content

Specific titles, for example "Sunsets, across 6 years", "Danilo, through the years", "A highlight from Pag Day&Night Festival". Language set in `config.yaml` like upstream's message templates.

## 4. Alternatives considered

| Option | Why not |
|---|---|
| Several windows with different event lists | More notifications per day, against D3 |
| A separate service next to upstream | Duplicates scheduling, dashboard and state |
| GPS based "trips" | Only 84 photos have GPS (D6) |
| One photo per notification | Does not meet the "some photos" goal (D4) |
| Immich Workflows (v3 preview) | Early stage, revisit later |

## 5. Testing

- **Visibility test first**: fixtures with locked and archived photos; assert they never reach the output of the new event types, including when Immich returns errors.
- **Contract tests** per Immich endpoint used.
- **Selection tests**: `varied` picks different types over many runs; cooldown respected; a set smaller than `min` is rejected; nothing eligible means no notification.
- **Replay simulation** (read only, against the real library): simulate 365 days; report notifications per day (at most 1), type mix, photos per set (3 to 6), repeat distances, and privacy violations (must be 0).
- Upstream's own test suite must keep passing.

## 6. Open questions

1. Final list of context themes (suggested above).
2. Which named people take part in `person_years`: all, or a chosen list?
3. Photos per set: default 4 within 3 to 6?
4. Language of titles: English or Italian?
5. Are the weights right (memories most often, others less)?

## 7. Status

draft, written 2026-11-25. Not approved. No code written.
