---
name: working-with-hal-apis
description: "Explores HAL+JSON APIs using the hal-walk CLI. Use when you need to discover, navigate, or integrate with a HAL API. Covers exploring API resources, reading link relation documentation, building traversal sessions, rendering traversal diagrams, and exporting deterministic path specs from recorded sessions."
---

<objective>
Use the `hal-walk` CLI to explore HAL+JSON APIs through HATEOAS link-following. You navigate by following hypermedia links — not by constructing URLs. The API tells you what's available at each position; you read relation docs to understand how to use each link, then follow it.

The workflow has two phases:
1. **Explore** — discover the API's capabilities by following links and reading relation docs
2. **Distill** — export a recorded traversal as a deterministic path spec that can run without inference
</objective>

<quick_start>
Start a session at the API root, then explore from there:

```bash
npx hal-walk start -s session.json https://agent-wiki.mikekelly321.workers.dev/
npx hal-walk position -s session.json           # see current position + available links
npx hal-walk describe -s session.json wiki:v1:pages  # read the relation docs (markdown)
npx hal-walk follow -s session.json wiki:v1:pages    # follow the link
```

Every command except `describe` outputs JSON to stdout. `describe` outputs the raw markdown relation documentation.
</quick_start>

<essential_principles>
**Follow your nose.** Never construct URLs manually. Every action starts from the links available at your current position. If you need to reach a resource, navigate there through links.

**Read before you act.** When you encounter an unfamiliar link relation, use `npx hal-walk describe` to fetch its documentation. The docs tell you the HTTP method, required input schema, and expected response. This is how you figure out what to send — not by guessing.

**CURIEs are namespaced relations.** Links like `wiki:v1:pages` are CURIEs (Compact URIs). The prefix `wiki:` expands to a documentation URL via a template declared at the API root (e.g. `wiki:v1:pages` expands to `.../rels/v1:pages`). `npx hal-walk describe` handles this expansion automatically.

**The session is a graph.** Every `follow` records a position (node) and transition (edge). You can jump back to any previous position with `goto` without re-fetching. The graph can be rendered as a Mermaid diagram or exported as a path spec.
</essential_principles>

<commands>
All commands require `-s, --session FILE`.

**`start URL`** — Begin session, GET the root resource.

**`position`** — Show current position, response, and available links.

**`goto POSITION_ID`** — Jump to a previous position (local, no HTTP request).

**`describe RELATION`** — Fetch relation documentation (outputs markdown, not JSON).

**`follow RELATION`** — Follow a link relation from the current position.

| Option | Description |
|--------|-------------|
| `--body JSON` | Request body (JSON string). Defaults method to POST if `--method` not set. |
| `--body-schema JSON` | JSON Schema for the body, from relation docs. Validates body before sending. If omitted, inferred from body. |
| `--uri-template-values JSON` | Variables for URI template expansion (RFC 6570), e.g. `'{"vid":"2"}'`. |
| `--headers JSON` | Custom request headers as JSON object. |
| `--header-schema JSON` | JSON Schema for headers. Validates headers before sending. If omitted, inferred from headers. |
| `-m, --method METHOD` | HTTP method override (e.g. PUT, DELETE). |
| `-n, --note TEXT` | Semantic breadcrumb — brief description of why this step is being taken. |

**`render`** — Generate Mermaid diagram of the session graph.

| Option | Description |
|--------|-------------|
| `-o, --output FILE` | Write to file instead of stdout. |

**`export`** — Export a deterministic path spec from the session graph.

| Option | Description |
|--------|-------------|
| `-o, --output FILE` | Write to file instead of stdout. |
| `--from POS_ID` | Start position (defaults to first). |
| `--to POS_ID` | End position (defaults to current). |

**`session-viewer`** — Serve a web UI for inspecting the session graph. Opens on a free port.
</commands>

<exploration_workflow>
**Step 1: Start at the root**

```bash
npx hal-walk start -s session.json https://agent-wiki.mikekelly321.workers.dev/
```

Read the output. Look at the `links` array — these are the relations you can follow.

**Step 2: Understand a relation**

Pick a relation from the links. Fetch its documentation:

```bash
npx hal-walk describe -s session.json wiki:v1:create-page
```

The markdown doc tells you:
- The HTTP method (POST, PUT, DELETE, GET)
- The input schema (what fields to send, which are required)
- An example request
- What the response looks like

**Step 3: Follow the relation**

Use what you learned from the docs to construct the follow command:

```bash
# For GET links (no body needed):
npx hal-walk follow -s session.json wiki:v1:pages \
  --note "List all pages to find existing content"

# For POST/PUT links (provide --body and --body-schema from the docs you read):
npx hal-walk follow -s session.json wiki:v1:create-page \
  --body '{"title": "My Page", "body": "Content here", "tags": ["demo"]}' \
  --body-schema '{"type":"object","properties":{"title":{"type":"string","description":"Page title"},"body":{"type":"string","description":"Page content (markdown)"},"tags":{"type":"array","items":{"type":"string"},"description":"Optional tags"}},"required":["title","body"]}' \
  --note "Create a new wiki page about demo content"

# For templated links (provide variables):
npx hal-walk follow -s session.json wiki:v1:search \
  --uri-template-values '{"q": "protocols"}' \
  --note "Search for pages about protocols"

# For PUT (explicit method override):
npx hal-walk follow -s session.json wiki:v1:edit-page \
  --method PUT \
  --body '{"body": "Updated content", "editNote": "Fixed typo"}' \
  --note "Fix typo in page content"
```

**Step 4: Repeat**

Each response reveals new links. Read the output, describe unfamiliar relations, follow the ones relevant to your task. Navigate deeper into the API graph.

**Step 5: Backtrack if needed**

```bash
npx hal-walk goto -s session.json p2
```

Jump back to any recorded position to explore a different branch.
</exploration_workflow>

<distillation>
Once you've completed a task through exploration, export the traversal as a deterministic path spec:

```bash
npx hal-walk render -s session.json    # visualize what you traversed
npx hal-walk export -s session.json    # export as a path spec
```

The path spec is a JSON array of steps with exact relations, methods, body schemas, example data, and notes. It captures the full workflow you discovered — including the contracts you learned and why you took each step — so it can be replayed by a simple runner with no inference.

To export a specific sub-path:

```bash
npx hal-walk export -s session.json --from p1 --to p5
```

This uses BFS to find the shortest path between two positions in the session graph.
</distillation>

<method_inference>
The `follow` command defaults to GET. When you provide `--body`, it defaults to POST. For other methods (PUT, DELETE), use `--method` explicitly.

How to determine the correct method:
1. Run `npx hal-walk describe` on the relation
2. Read the "Method" section of the markdown
3. Use `--method` if it's not GET or POST

Common patterns:
- `create-*` relations → POST (inferred from `--body`)
- `edit-*` relations → PUT (use `--method PUT`)
- `delete-*` relations → DELETE (use `--method DELETE`)
- Everything else → GET (default)
</method_inference>

<schema_and_notes>
**Capture your inference.** When `describe` output includes a JSON Schema for the input, pass it via `--body-schema` to preserve the contract in the session. This captures what you learned — required vs optional fields, types, descriptions — so it's preserved in exports. If you omit `--body-schema`, a basic schema is auto-inferred from the body data, but this loses optionality info and descriptions.

The body is validated against the schema before sending. If they don't match, the command fails immediately — this catches bugs in your data construction.

**Leave breadcrumbs.** Always provide `--note` with a brief description of why you're taking each step. These notes are recorded on transitions and appear in exported path specs and the session viewer. They make your traversal self-documenting — someone reading the export understands not just *what* happened but *why*.
</schema_and_notes>

<success_criteria>
Exploration is successful when:
- You navigated the API entirely through link-following (no hardcoded URLs)
- You used `describe` to understand unfamiliar relations before acting
- The session file contains a complete record of your traversal

Distillation is successful when:
- `npx hal-walk render` produces a valid Mermaid graph showing the traversal
- `npx hal-walk export` produces a path spec with concrete steps, relations, methods, body schemas, and notes
- The path spec could be executed by a runner without any LLM inference
- Each step that sends data has a `bodySchema` capturing the contract and a `note` explaining intent
</success_criteria>
