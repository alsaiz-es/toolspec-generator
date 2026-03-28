# Demo: MusicBrainz API

Generation of a complete ToolSpec descriptor for the [MusicBrainz API](https://musicbrainz.org/doc/MusicBrainz_API) using the toolspec-generator skill in Claude Desktop.

## Prompt

```
Genera un ToolSpec completo para la API de MusicBrainz.
Documentación en https://musicbrainz.org/doc/MusicBrainz_API
```

## What the skill did

### 1. Read all API documentation

The skill fetched and read the full MusicBrainz API docs, following links to sub-pages for search, lookup, browse, rate limiting, and authentication.

### 2. Mapped the entity model

Identified the complete entity hierarchy before writing any tools:

```
Artist -> Release Group -> Release -> Medium -> Track -> Recording
```

Plus supporting entities: Area, Event, Genre, Instrument, Label, Place, Series, Work, URL, Collection.

### 3. Catalogued all endpoints

Mapped the three core API operations across all entity types:

| Operation | Count | Description |
|-----------|-------|-------------|
| Lookup    | 17    | 13 core entities + url-by-resource, discid, isrc, iswc |
| Search    | 15    | All Lucene-indexed types (annotation, area, artist, cdstub, event, instrument, label, place, recording, release, release-group, series, tag, url, work) |
| Browse    | 13    | area, artist, collection, event, genre, instrument, label, place, recording, release, release-group, series, work |
| Special   | 1     | list_all_genres |
| **Total** | **46**| |

### 4. Validated URL patterns

Tested real API calls to verify that `base_url` + `endpoint.path` formed valid URLs. Rate limiting (503) and expected 404s confirmed the patterns were correct.

### 5. Generated the complete descriptor

The output included:

- **46 tools** with full parameter schemas, response schemas, error codes, `when_to_use` guidance, and `estimated_duration_seconds`
- **5 workflow examples** showing real multi-step chains (artist discography, ISRC identification, live bootleg search, label catalog browsing, Spotify URL resolution)
- **Knowledge layer** with `system_context` documenting 10 critical API quirks, 2 named workflows, and a glossary of 16 domain terms
- **Service metadata** with auth requirements (User-Agent mandatory), rate limits (50 req/min), and capabilities

## Output

The generated descriptor is available as the reference example: [`references/musicbrainz-example.json`](../references/musicbrainz-example.json)

## Validate and install

```bash
cd your-llm-toolspec-project
npx toolspec validate musicbrainz.toolspec.json
npx toolspec install musicbrainz.toolspec.json
# Restart Claude Desktop
```

## Result

The MusicBrainz ToolSpec is running as an MCP server in Claude Desktop, providing all 46 tools for music metadata queries.
