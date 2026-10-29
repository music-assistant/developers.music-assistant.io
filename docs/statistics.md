# Statistics API

The `statistics/*` commands return listening statistics built from the play history. Every
command needs the `library:read` scope and only covers the plays of the authenticated user.

These commands require API schema version 100 or newer. Older servers may expose an earlier
version of the statistics API with incomplete results.

## Commands

| Command | Returns |
|---|---|
| `statistics/play_history` | `list[ItemMapping]`, most recent first |
| `statistics/top_items` | `list[TopItemResult]` |
| `statistics/genre_distribution` | `list[DistributionItem]` |
| `statistics/decade_distribution` | `list[DistributionItem]` |
| `statistics/play_count` | `int` |
| `statistics/plays_over_time` | `list[TimeSeriesPoint]` |
| `statistics/listening_activity` | `list[HeatmapPoint]` |
| `statistics/listening_time` | `list[ListeningTimeItem]` |

The result models live in
[`music_assistant_models.statistics`](https://github.com/music-assistant/models/blob/main/music_assistant_models/statistics.py).
The command docstrings in the server describe every parameter.

```python
from music_assistant_models.enums import MediaType

top_artists = await client.send_command(
    "statistics/top_items",
    media_type=MediaType.ARTIST,
    period="month",
    limit=10,
    timezone="Europe/Berlin",
)
for entry in top_artists:
    print(entry["item"]["name"], entry["play_count"])
```

## Periods and time zones

Most commands take a `period`:

| Period | Covers |
|---|---|
| `today` | Since local midnight |
| `week` | The last 7 days |
| `month` | The last 30 days |
| `year` | The last 365 days |
| `all_time` | Everything |

Pass the user's IANA `timezone` (for example `Europe/Berlin`). It sets where `today` starts and
how plays fall into hours, days and weeks. It defaults to `UTC`, so a client that leaves it out
shows buckets that are shifted for most users. An unknown period or time zone fails with
`InvalidDataError`.

`limit` is capped at 500 for the ranking and distribution commands.

## Result details

- **Items** in `play_history` and `top_items` are `ItemMapping` references. Their `available`
  flag reflects whether the item can still be played today, because the history outlives the
  library.
- **Albums and artists** count the plays of their tracks: `play_count` and `top_items` count the
  album's tracks for an album and the tracks an artist is credited on for an artist.
- **`plays_over_time`** returns a complete series, oldest first. Every bucket from the start of the
  period (or the first play, for `all_time`) through the current one is included, with `0` for
  buckets without plays. It returns an empty list when the period holds no plays at all. The
  `timestamp` format depends on `granularity`:

    | Granularity | Timestamp |
    |---|---|
    | `hour` | `2026-10-29 14:00:00` |
    | `day` | `2026-10-29` |
    | `week` | `2026-10-26`, the Monday the week starts on |
    | `month` | `2026-10` |

- **`listening_activity`** returns one `HeatmapPoint` per hour of day (`0`-`23`) and weekday
  (`0` is Monday) that has plays. Missing cells mean no plays.
- **`listening_time`** sums the durations of completed plays of library tracks, in minutes. With
  `group_by="artist"` each result carries the artist in `item`, so two artists with the same name
  stay separate. With `group_by="genre"`, `item` is empty.
- **Genre and decade** results only count library tracks with a known genre or album year.
