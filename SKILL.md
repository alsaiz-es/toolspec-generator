---
name: toolspec-generator
description: "Generate Open LLM Tool Specification (ToolSpec) descriptors from API documentation. Use this skill whenever the user wants to create a toolspec.json file, convert API documentation to ToolSpec format, generate LLM tool descriptors from OpenAPI/Swagger specs, or make any API consumable by LLMs via ToolSpec. Also trigger when the user mentions 'toolspec', 'tool specification', 'LLM tool descriptor', or wants to expose an API as MCP tools. This skill reads API docs (URLs, OpenAPI specs, or plain text descriptions) and outputs a valid toolspec.json that can be installed as an MCP server proxy."
---

# ToolSpec Generator

Generate valid ToolSpec descriptors from API documentation. ToolSpec is a vendor-agnostic JSON format that describes APIs for LLM consumption — think "OpenAPI for LLMs" with three layers: Service (how to connect), Tools (what to call), and Knowledge (how to reason).

## When to read reference files

Before generating a descriptor, read these files from the `references/` directory:

- **Always read** `toolspec-schema-v0.1.json` — the formal JSON Schema. Your output must validate against this.
- **Always read** `musicbrainz-example.json` — the reference example. Match this format and style.
- **Read if first time** `lessons-learned.md` — practical lessons from real ToolSpec generation. Prevents common mistakes.

## Generation workflow

### Step 1: Gather API information

If the user provides a URL, fetch and read ALL the documentation. If they provide an OpenAPI spec, parse it completely. If they describe the API informally, ask clarifying questions about:

- Base URL
- Authentication method (API key, OAuth, none, custom headers)
- Rate limits
- Complete list of available endpoints

Read every documentation page, not just the overview. Follow links to sub-pages (search docs, examples, rate limiting, etc.). An incomplete read produces an incomplete descriptor.

### Step 2: Map the entity model

Before writing any tools, understand the API's data model:

1. **Identify all entities** (e.g., Artist, Release Group, Release, Recording)
2. **Map the hierarchy** — what links to what, parent-child relationships, what IDs chain between entities
3. **Identify the navigation patterns** — how a user gets from entity A to entity B. If A → B → C, you need tools for every hop. Missing a level means the LLM can't complete the chain.
4. **Document this as a mental model** before proceeding

For example, MusicBrainz has: Artist → Release Group → Release → Medium → Track → Recording. Skipping "Release Group" means you can't distinguish between "the album" and "a specific edition". Skipping "Release lookup with recordings" means you can't get a tracklist.

This step prevents the most common failure: generating tools that cover some endpoints but leave gaps in the navigation chain.

### Step 3: Document ALL endpoints

Generate a tool for EVERY endpoint the API exposes. This is not optional. A ToolSpec is a complete descriptor, not a summary. If the API has 15 endpoints, the ToolSpec has 15 tools. If it has 50, it has 50.

The consumer (the LLM client) decides which tools to use at runtime. The descriptor's job is to make ALL of them available, correctly documented, and usable. Omitting endpoints is like shipping an OpenAPI spec with missing paths — it's broken by definition.

For each tool, follow these rules:

**Naming:**
- `name`: lowercase with underscores, descriptive and unique (`search_artists`, `lookup_artist`, `browse_release_groups`)
- Prefix with the operation type when the API has multiple operations per entity: `search_`, `lookup_`, `browse_`, `create_`, `update_`, `delete_`

**Descriptions — document REAL behavior:**
- `description`: What the tool does, including quirks, constraints, and non-obvious behavior. Document how the API ACTUALLY works, not what the spec claims.
- `when_to_use`: Natural language guidance from the user's perspective. This is what the LLM reads to decide which tool to call.
  - GOOD: "When the user wants to find songs by a specific artist or discover new music."
  - BAD: "Calls the /search endpoint with type=recording."

**Endpoint:**
- `method` + `path`. Path parameters use `{param}` syntax.
- Do NOT bake query parameters into the path (e.g., don't use `/artist?fmt=json`). Instead, add them as parameters with defaults.

**Parameters:**
- Proper JSON Schema with `type`, `description`, `enum`, `default`, `format`
- Mark `required` fields explicitly — if the API fails without a parameter, it's required regardless of what the docs say
- Path parameters (matching `{param}` in endpoint.path) must appear in parameters
- Parameters that must always have a fixed value (like `fmt=json`) should have `enum` with one value and be in `required`

**Response schemas:**
- Document ALL fields the API returns, especially IDs that chain to other tools
- If tool A returns an `id` that tool B needs, make it explicit in both schemas
- Include fields like `disambiguation`, `barcode`, `cover-art-archive` — anything the API returns

**Errors:**
- Actionable descriptions with recovery guidance:
  - GOOD: `"Artist not found. Verify the MBID is correct or try searching by name first."`
  - BAD: `"Not found"`

**Metadata:**
- `estimated_duration_seconds`: With headroom above typical response time
- `idempotent`: true for GET/safe operations, false for mutations

### Step 4: Write comprehensive examples

Write examples that cover:
1. The primary end-to-end workflow (search → navigate → detail)
2. Every major entity type in the API
3. At least one example per navigation pattern

Use `${step_N.field}` syntax for referencing previous outputs. Each step has a `note` explaining why.

Examples must reflect the REAL chain of calls needed. If getting a tracklist requires search → browse release groups → browse releases → lookup release, show all 4 steps. Don't skip intermediate steps.

### Step 5: Knowledge layer (when needed)

Include the knowledge layer when:
- The domain has concepts the LLM wouldn't know from general training
- Results require interpretation (what does a metric mean? what's normal vs abnormal?)
- There are non-obvious patterns (entity hierarchies, naming conventions, data quirks)

Skip it when the domain is self-explanatory (basic CRUD, well-known public APIs).

### Step 6: Self-review checklist

Before presenting the output, verify:

1. **Completeness**: Does every API endpoint have a corresponding tool? If not, why?
2. **Entity coverage**: Can the LLM navigate from any entity to any related entity without gaps?
3. **URL validity**: Does `base_url` + each `endpoint.path` form a valid, callable URL?
4. **No baked-in query params**: Are all query parameters defined as parameters, not embedded in the path?
5. **Required fields**: Are all parameters that cause errors when omitted marked as `required`?
6. **Response schemas**: Do they include IDs and fields needed for chaining?
7. **Quirks documented**: Are non-obvious behaviors explained in descriptions?
8. **Examples complete**: Do workflows show the full chain without skipping intermediate calls?
9. **Rate limit documented**: Is `capabilities.rate_limit` accurate?
10. **Auth complete**: Are all required headers (including non-standard ones like User-Agent) documented?

### Step 7: Present and iterate

Show the complete JSON to the user. Suggest they:
1. Save it as `<service-name>.toolspec.json`
2. Validate: `npx toolspec validate <file>`
3. Install: `npx toolspec install <file>`
4. Restart Claude Desktop and test

If the user reports issues after testing, update the descriptor to reflect real behavior.

## Key principles

**Complete, not curated.** Document every endpoint. The descriptor is a complete contract, not a best-of selection. The consumer decides what to use.

**Map the model first, then the endpoints.** Understanding entity relationships prevents gaps in navigation chains. If you can't draw the entity hierarchy, you'll miss tools.

**Document reality, not documentation.** If the official docs say a field is optional but the server returns 500 without it, mark it as required and explain why.

**`when_to_use` drives tool selection.** This is what makes ToolSpec different from OpenAPI. An LLM reads this field to decide which tool to call. Invest time here.

**Response schemas enable chaining.** Without documented response fields, the LLM can't extract values for the next call.

**No query params in paths.** Always define them as parameters with defaults. The executor builds the URL.

**Test the URLs.** `base_url` + `endpoint.path` must be a real, callable URL. This is the #1 source of bugs.
