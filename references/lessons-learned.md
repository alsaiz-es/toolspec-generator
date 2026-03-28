# ToolSpec generation — lessons learned

These are real problems encountered while building and testing ToolSpec descriptors. Apply these lessons when generating new descriptors.

## Lesson 1: Official docs lie, document real behavior

The MusicBrainz API docs say the `fmt` parameter is optional and defaults to XML. In practice, omitting it returns XML that breaks JSON-expecting clients silently. The ToolSpec must document the real behavior: mark `fmt` as required with `"enum": ["json"]` so the LLM always sends it.

Similarly, the docs describe `User-Agent` as "recommended" but the server returns HTTP 403 without one. Document it as mandatory in the auth scheme description.

When converting from API documentation, always flag fields where "optional" might actually be "required in practice". Common patterns:
- Parameters documented as optional that cause errors when omitted
- Default values that the server doesn't actually apply
- Headers described as "recommended" that are actually enforced
- Enum values that are documented but rejected by the server

## Lesson 2: when_to_use drives tool selection

The LLM uses `when_to_use` to decide which tool to call. It must be written from the user's perspective, not the API's:

GOOD: "When the user wants to find songs by a specific artist or discover new music."
BAD: "Searches the recording endpoint with Lucene query syntax."

The `description` explains WHAT the tool does. The `when_to_use` explains WHEN to pick it. Both are needed.

## Lesson 3: Response schemas enable chaining

If tool A returns an `id` that tool B needs as input, the response schema of tool A must document that `id` field. Without it, the LLM doesn't know what to extract for the next call.

The `examples` section with `${step_N.field}` references teaches the LLM the chaining pattern, but the response schema is what tells it which field to grab.

## Lesson 4: GET vs POST parameter routing matters

The ToolSpec executor routes parameters based on HTTP method:
- **GET/DELETE**: non-path parameters become query string (`?status=available`)
- **POST/PUT/PATCH**: non-path parameters become the JSON body

Path parameters (matching `{param}` in the endpoint path) are always extracted from the input and interpolated into the URL, regardless of method.

This means:
- For GET endpoints, every parameter must be serializable as a string in a query parameter
- For POST endpoints, nested objects and arrays are fine (they go as JSON body)
- Path parameters must appear in both the `endpoint.path` template AND the `parameters` object

## Lesson 5: Custom headers need explicit documentation

If an API requires a custom User-Agent header (like MusicBrainz), or a non-standard auth header, or a format parameter (like `fmt=json`), document it explicitly. The SDK cannot guess these. Options:

1. Put fixed headers in the auth scheme if they're auth-related
2. Document format requirements (like `fmt=json`) as fixed query parameters in the tool endpoint description
3. For User-Agent requirements, note it in the service description AND in the capabilities section

## Lesson 6: Rate limits are critical for the SDK

The ToolSpec SDK uses `capabilities.rate_limit` to throttle requests. If you omit it or get it wrong:
- Too high: the API blocks you (MusicBrainz bans IPs that exceed 1 req/sec)
- Too low: the user experience degrades unnecessarily

Always document the real rate limit. If unknown, set a conservative default and note it.

## Lesson 7: estimated_duration_seconds prevents timeouts

The executor uses this to set request timeouts. A search endpoint that takes 5 seconds needs `"estimated_duration_seconds": 8` (with headroom). If the timeout is too low, valid responses get cut off.

## Lesson 8: Document ALL endpoints, no exceptions

A ToolSpec must cover every endpoint the API exposes. It's a complete descriptor, not a curated selection. Omitting endpoints creates gaps — if a user needs a tool that exists in the API but not in the descriptor, the ToolSpec is broken.

The original mistake was limiting to "5-10 tools" to avoid context bloat. This caused two independent ToolSpec generations for MusicBrainz to produce different subsets — one missing `lookup_release` (no tracklists), the other missing `browse_release_groups` (no album-level navigation). Both were incomplete. Both were wrong.

The consumer decides which tools to use at runtime. The descriptor's job is to make all of them available. If the API has 50 endpoints, the ToolSpec has 50 tools — each properly documented.

## Lesson 9: Test with real API calls

After generating a descriptor, test the key paths:
- Does `base_url` + `endpoint.path` form a valid URL?
- Do GET requests with query parameters work?
- Do POST requests with JSON bodies work?
- Do path parameters get interpolated correctly?
- Do the documented errors actually match what the API returns?

## Lesson 10: The knowledge layer is only for non-obvious domains

For a simple CRUD API (e.g., a todo list), no knowledge layer is needed — any LLM understands the domain. For MusicBrainz, the knowledge layer is essential because the LLM needs to understand the entity hierarchy (Artist → Release Group → Release → Medium → Track → Recording), the difference between lookup/browse/search operations, and quirks like the 25-entity cap on lookup inc= results. For a specialized diagnostic API, it would be even more critical — the LLM would need to know analysis patterns, what metrics mean, and how to interpret results.

Rule of thumb: if you need to explain to a junior developer what the results mean, you need a knowledge layer. If the results are self-explanatory, skip it.

## Lesson 11: Map the entity model BEFORE writing tools

Two independent generations of a MusicBrainz ToolSpec produced different tools because neither mapped the full entity hierarchy first. MusicBrainz has: Artist → Release Group → Release → Medium → Track → Recording. One generation skipped Release Group, the other skipped Recording lookup. Both were incomplete.

Before writing any tools, draw the entity graph:
- What are the entities?
- What links to what?
- What IDs chain between calls?
- Can the LLM navigate from any entity to any related entity without gaps?

If you can't traverse the full graph with the tools you've defined, you're missing tools.

## Lesson 12: Never bake query parameters into endpoint paths

Writing `"path": "/artist?fmt=json"` breaks the executor because it concatenates additional query params with `?`, producing `base/artist?fmt=json?query=radiohead` (double question mark). Always define query parameters as tool parameters with defaults:

```json
"parameters": {
  "type": "object",
  "properties": {
    "fmt": {
      "type": "string",
      "enum": ["json"],
      "default": "json",
      "description": "Response format. Always use json."
    }
  },
  "required": ["fmt"]
}
```
