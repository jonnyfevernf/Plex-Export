# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Project Does

Plex Export is a two-phase tool that produces a static, shareable HTML page from a Plex Media Server library:

1. **Data extraction (PHP CLI):** `cli.php` queries the Plex HTTP API (XML), downloads thumbnails via the Plex transcoder, and writes `plex-data/data.js` — a JS file that assigns a large JSON blob to `var raw_plex_data`.
2. **Static frontend (HTML + JS):** `index.html` fetches `plex-data/data.js` via AJAX, evals it to unpack `raw_plex_data`, and drives all UI through the `PLEX` singleton in `assets/js/plex.js`.

There is no build system, no package manager, and no test suite.

## Running the Exporter

```bash
# Basic usage (Plex on localhost)
php cli.php

# Remote Plex server
php cli.php -plex-url=http://other-machine.local:32400

# Authenticated (Home/managed users)
php cli.php -token=<X-Plex-Token>

# Export specific sections by name or numeric ID
php cli.php -sections="Movies,TV Shows"

# All parameters with defaults
php cli.php \
  -plex-url=http://localhost:32400 \
  -data-dir=plex-data \
  -sections=all \
  -token= \
  -thumbnail-width=150 \
  -thumbnail-height=250 \
  -sort-skip-words=a,the,der,die,das
```

PHP requirements: `simplexml` extension enabled, `allow_url_fopen = On`, `plex-data/` directory must be writable. Memory limit is set to 512 MB inside the script.

## Architecture

### Data flow

```
cli.php → Plex HTTP API (XML) → parse movies/shows → download thumbnails
       → json_encode → plex-data/data.js (var raw_plex_data = {...})

index.html (browser) → $.get("plex-data/data.js") → eval() → PLEX.load_data()
                     → PLEX.run() → renders sidebar + grid UI
```

### `cli.php` internals

- `load_all_sections()` — hits `/library/sections`, returns only `movie` and `show` types (other types are skipped with a warning).
- `load_items_for_section()` — dispatches to `load_data_for_movie()` or `load_data_for_show()`.
- `load_data_for_movie()` — fetches `/library/metadata/<key>` for full metadata (genres, directors, cast) beyond what the section listing provides.
- `load_data_for_show()` — recursively fetches seasons then episodes via `/library/metadata/<key>/children`.
- `generate_item_thumbnail()` — uses the Plex `/photo/:/transcode` endpoint to resize; skips if the file already exists, so re-runs are incremental.
- Sort indexes are pre-computed PHP-side (`title_asc/desc`, `release_asc/desc`, `rating_asc/desc`, `addedAt_asc/desc`) and embedded in the JSON so the browser never has to sort.
- The file contains a bundled `JavaScriptPacker` / `ParseMaster` class (a PHP port of Dean Edwards' JS Packer). It is currently **disabled** — `$packed_js = $raw_js` on line 215 bypasses it.

### `assets/js/plex.js` — `PLEX` singleton

State lives directly on the `PLEX` object: `current_section`, `current_sort_key/order`, `current_genre`, `current_director`, `current_seen`. All DOM references are cached as `PLEX._*` properties during `PLEX.run()`.

Filter chain in `display_items()`: seen → text search → genre → director, applied in that order over `current_section.items`. The sorted display order comes from the pre-built `current_section.sorts[key+"_"+order]` array (indexes into the items map).

Hash routing: `#<section_id>` or `#<section_id>/<item_id>` — set on section/item open, parsed on load.

Keyboard shortcuts: `j`/`k` navigate items in the popup; `Esc`/`x` closes it.

### `assets/js/utils.js`

Standalone helpers: `episode_tag(season, episode)` → `S01E03`, `number_format()`, `inflect()`, `hl_bytes_to_human()`.

### `plex-data/` directory

Git-ignored for `*.js`, `*.png`, `*.jpeg` — all generated output (data file + thumbnails) lives here and is not committed.

## Key Conventions

- **No framework or build step** on the frontend. jQuery 1.4.4 is the only dependency (bundled at `assets/js/jquery.1.4.4.min.js`). `.live()` is used throughout (deprecated in newer jQuery — do not upgrade jQuery without updating all event bindings).
- **`cli.php` uses global state** (`$options`, `$context`) accessed via `global` inside functions — this is intentional for the single-script CLI design.
- **Thumbnails are cached by key** (`thumb_<ratingKey>.jpeg`). To force re-download, delete the file.
- **Only `movie` and `show` section types are supported.** Adding support for another type requires a new parser function and a new case in `load_items_for_section()`.
- The `data.js` output is loaded via `eval()` in the browser — the file assigns to `var raw_plex_data` and is not a JSON file.
