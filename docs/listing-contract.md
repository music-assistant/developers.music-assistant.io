# Listing contract

Music Assistant listing commands share a contract for sorting, filtering and paging. The server
is authoritative for which sort fields each listing supports; clients should discover those
options instead of assuming that every listing offers the same fields.

## Discover sort options

Call `music/sort_options` with a `ListingType`. The response is a list of `SortOptionInfo` values;
the default option is first. Each option identifies its `SortField`, whether direction is
supported, and its default direction when applicable.

```python
from music_assistant_models.enums import ListingType, SortDirection, SortField

options = await client.music.get_sort_options(ListingType.LIBRARY_TRACKS)
tracks = await client.music.get_library_tracks(
    limit=100,
    offset=0,
    sort_field=SortField.NAME,
    sort_direction=SortDirection.ASC,
)
```

Typed sorting is available on library listings and supported detail listings, including album,
artist, genre, playlist and podcast contents, browse results, similar tracks and media versions.
Pass `sort_direction` only when the selected option supports it. Omit either sort argument to use
the listing's default.

## Pagination and filtering

Listing methods accept `limit` and `offset` where paging is supported. `search` and listing-specific
filters can be combined with paging and sorting. In particular, `music/playlists/playlist_tracks`
and `music/podcasts/podcast_episodes` return one page as a list, with a default limit of 500 and an
offset of 0. Request subsequent pages by advancing `offset`; these commands no longer produce
chunked results.

The Python client exposes these as ordinary list-returning methods:

```python
page_size = 100
offset = 0
while page := await client.music.get_podcast_episodes(
    podcast_id,
    provider,
    limit=page_size,
    offset=offset,
):
    process(page)
    offset += len(page)
```

## Random ordering

For listings that offer `SortField.RANDOM`, pass the same integer `random_seed` on every page in a
single listing load. This keeps the order stable while paging. Use a new seed to request a new
random order. A seed does not freeze the underlying data, and the server remains authoritative for
whether a listing offers random sorting. `random_seed` requires API schema 98; the Python client
checks this minimum when the argument is set.

```python
seed = 4217
first_page = await client.music.get_library_tracks(
    limit=100,
    offset=0,
    sort_field=SortField.RANDOM,
    random_seed=seed,
)
second_page = await client.music.get_library_tracks(
    limit=100,
    offset=100,
    sort_field=SortField.RANDOM,
    random_seed=seed,
)
```

See the server's [random listing notes](https://github.com/music-assistant/server/blob/dev/DEVELOPMENT.md#listing-random-order)
for details.

## Playback sorting

`player_queues/play_media` accepts `sort_field` and `sort_direction` to order album or playlist
tracks before applying `start_item`. Use the same sort field offered by the corresponding listing.
The deprecated `sort_by` string is retained for older clients only.

## Migration

- Replace `order_by` with `sort_field` and, where supported, `sort_direction`.
- Replace playback `sort_by` with `sort_field` and `sort_direction`.
- Replace chunked playlist-track and podcast-episode consumption with repeated paged calls using
  `limit` and `offset`; each response is a list containing at most one page.
- Discover supported fields with `music/sort_options` instead of hard-coding a sort list.
