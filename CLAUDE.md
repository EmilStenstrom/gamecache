# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

MyBGG is a Python-based application that creates a searchable website for BoardGameGeek collections. It downloads board game data from BGG, creates a SQLite database, and hosts it as a static website via GitHub Pages. The frontend uses vanilla JavaScript with SQL.js to query the database client-side.

## Essential Commands

### Development Workflow
```bash
# Install dependencies
pip install -r scripts/requirements.txt

# Validate configuration and setup
python scripts/validate_setup.py

# Generate database (with caching for faster subsequent runs)
python scripts/download_and_index.py --cache_bgg

# Generate database without GitHub upload (for testing)
python scripts/download_and_index.py --cache_bgg --no_upload

# Generate with debug logging
python scripts/download_and_index.py --debug --cache_bgg

# Enable automated hourly updates via GitHub Actions
python scripts/enable_hourly_updates.py

# Test website locally
python -m http.server
# Then open http://localhost:8000
```

### Configuration
The `config.ini` file at the project root contains user-specific settings:
- `title`: Website title
- `bgg_username`: BoardGameGeek username
- `github_repo`: GitHub repository in format `owner/repo`

## Architecture

### Data Flow Pipeline
1. **Download** (`scripts/mybgg/downloader.py`): Orchestrates data fetching from BGG
2. **BGG Client** (`scripts/mybgg/bgg_client.py`): Handles BGG XML API requests with retry logic and rate limiting
3. **Models** (`scripts/mybgg/models.py`): Transforms raw BGG data into `BoardGame` objects
4. **SQLite Indexer** (`scripts/mybgg/sqlite_indexer.py`): Creates database with game data + extracted dominant colors from images
5. **GitHub Integration** (`scripts/mybgg/github_integration.py`): Uploads compressed database to GitHub Releases using OAuth Device Flow

### Frontend Architecture
- **index.html**: Template structure with Material Symbols icons
- **app-sqlite.js**: Client-side app that:
  - Loads `config.ini` to get GitHub repo
  - Fetches gzipped SQLite database from GitHub Releases
  - Uses SQL.js to query the database in-browser
  - Implements filtering, sorting, pagination, and search

### Key Design Patterns

**BGG API Interaction**: The `BGGClient` uses exponential backoff with jitter for retries. BGG API returns "202 Accepted" for collection requests requiring queuing - the client polls until data is ready.

**Expansions Handling**: Expansions are stored as nested objects within games. The `BoardGame.calc_num_players()` method merges player counts from base game + all expansions, marking sources as "official", "best", "recommended", or "expansion".

**Color Extraction**: During indexing, the `SqliteIndexer` fetches game images and uses colorgram to extract a dominant color (avoiding too-dark/too-light colors). This color is stored as RGB string for UI theming.

**Authentication**: Uses GitHub OAuth Device Flow (not personal access tokens). Token is cached in `~/.mybgg/token.json` and validated before each use. For CI/CD, the `MYBGG_GITHUB_TOKEN` environment variable bypasses OAuth.

## Important Implementation Details

### BGG API Constraints
- Rate limiting: 429 errors trigger 30-second wait
- Batch game details in chunks of 20 IDs to avoid URI length limits
- Collection requests may return 202 and require polling
- XML responses use declxml library for parsing

### Database Schema
The SQLite database has a single `games` table with JSON-encoded arrays for:
- `categories`, `mechanics`, `tags`, `previous_players`, `expansions`
- `players` array contains tuples of `[player_count, type]` where type is "official", "best", "recommended", or "expansion"
- `color` field stores RGB as string "R, G, B"

### Player Count Logic
Player counts are aggregated from three sources in `BoardGame.calc_num_players()`:
1. Community-voted "suggested_numplayers" from BGG (marked as "best" or "recommended")
2. Official min/max players from publisher (marked as "official")
3. Player counts enabled by expansions (marked as "expansion")

All three are merged and displayed in the UI with different visual treatments.

### GitHub Release Strategy
- Database is uploaded to a release with tag `database` (configurable)
- Asset name is `mybgg.sqlite.gz` (configurable)
- The frontend always fetches from `releases/latest/download/`
- Old assets are deleted before uploading new versions

### Python Version Compatibility
Tested with Python 3.8-3.12. For Python 3.13+, regenerate dependencies:
```bash
pip install pip-tools
pip-compile scripts/requirements.in -o scripts/requirements.txt
```

## Common Development Scenarios

### Adding New BGG Data Fields
1. Update XML parsing in `bgg_client.py` (`_games_list_to_games()` or `_collection_to_games()`)
2. Add field to `BoardGame.__init__()` and `todict()` in `models.py`
3. Update SQLite schema in `sqlite_indexer.py` (`_init_database()`)
4. Update INSERT statement in `sqlite_indexer.py` (`add_objects()`)
5. Update frontend query and rendering in `app-sqlite.js`

### Modifying Filters
Frontend filters are dynamically generated from database aggregations. To add a filter:
1. Add facet container to `index.html`
2. Implement aggregation query in `app-sqlite.js`
3. Add filter UI generation logic
4. Update `applyFilters()` to include new filter in WHERE clause

### Testing Changes Locally Without Upload
Always use `--no_upload` flag when testing database generation:
```bash
python scripts/download_and_index.py --cache_bgg --no_upload
```

Then test the generated `mybgg.sqlite.gz` by:
1. Manually copying it to the expected release location, OR
2. Temporarily modifying `app-sqlite.js` to load from a local path

## Troubleshooting

### BGG Data Issues
- **No games imported**: Check BGG username, ensure collection is public, verify games are marked as "owned"
- **Slow downloads**: Use `--cache_bgg` flag (creates `mybgg-cache.sqlite`)
- **Rate limiting**: The client automatically handles 429 errors with exponential backoff

### Authentication Issues
- OAuth token stored in `~/.mybgg/token.json`
- Token validation happens before each upload attempt
- For CI/CD, set `MYBGG_GITHUB_TOKEN` environment variable
- Token requires `public_repo` scope for creating releases

### Database Generation
- First run is slow (5-15 min depending on collection size)
- Image fetching for color extraction can fail - falls back to white (255, 255, 255)
- Duplicate games are deduplicated by ID in `download_and_index.py`

## File Organization
- `scripts/`: All Python backend code
  - `scripts/mybgg/`: Core library modules
  - `scripts/download_and_index.py`: Main entry point
  - `scripts/validate_setup.py`: Configuration validator
  - `scripts/enable_hourly_updates.py`: GitHub Actions setup
- Root directory: Frontend HTML, CSS, JS
- `.github/workflows/index.yml`: Hourly automated updates
