# ToolSpec Generator

A Claude Desktop skill that generates [ToolSpec](https://github.com/alsaiz-es/llm-toolspec) descriptors from API documentation.

## What it does

Give it API documentation (a URL, an OpenAPI spec, or a plain text description) and it generates a complete `toolspec.json` descriptor — ready to install as an MCP proxy with one command.

## Install

Clone the repo and add the `SKILL.md` and `references/` directory to your Claude Desktop project.

## Usage

Open any conversation in Claude Desktop and say:

```
Generate a ToolSpec for the MusicBrainz API.
Documentation at https://musicbrainz.org/doc/MusicBrainz_API
```

The skill will:

1. Read the full API documentation
2. Map the entity model and identify all endpoints
3. Generate a complete descriptor with every endpoint documented
4. Include semantic `when_to_use` guidance for each tool
5. Add workflow examples showing how tools chain together

Save the output as `<service>.toolspec.json`, then:

```bash
npx toolspec validate <file>
npx toolspec install <file>
# Restart Claude Desktop
```

## What makes it different from writing descriptors by hand

- **Documents ALL endpoints**, not a curated subset. A ToolSpec is a complete contract.
- **Maps the entity model first** to ensure no navigation gaps (e.g., Artist → Release Group → Release → Recording).
- **Documents real behavior**, not what the official docs claim. If a field marked "optional" causes a 500 when omitted, the descriptor says it's required.
- **Writes `when_to_use` from the user's perspective**, not the API's. This is the field LLMs use to decide which tool to call.
- **No query params baked into paths** — parameters are always defined separately so the executor builds URLs correctly.

## Skill contents

```
toolspec-generator/
├── SKILL.md                              # Skill instructions
├── references/
│   ├── toolspec-schema-v0.1.json         # JSON Schema for validation
│   ├── musicbrainz-example.json          # Reference example
│   └── lessons-learned.md                # Real-world lessons from ToolSpec generation
└── demos/
    └── musicbrainz.md                    # Full generation walkthrough
```

## Related

- [llm-toolspec](https://github.com/alsaiz-es/llm-toolspec) — The spec, SDK, and CLI for consuming ToolSpec descriptors

## License

Apache 2.0
