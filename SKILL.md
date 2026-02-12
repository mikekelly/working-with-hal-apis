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
npx hal-walk start -s session.json http://localhost:3000/
npx hal-walk position -s session.json           # see current position + available links
npx hal-walk describe -s session.json wiki:pages  # read the relation docs (markdown)
npx hal-walk follow -s session.json wiki:pages    # follow the link
```

Every command except `describe` outputs JSON to stdout. `describe` outputs the raw markdown relation documentation.
</quick_start>

<essential_principles>
**Follow your nose.** Never construct URLs manually. Every action starts from the links available at your current position. If you need to reach a resource, navigate there through links.

**Read before you act.** When you encounter an unfamiliar link relation, use `npx hal-walk describe` to fetch its documentation. The docs tell you the HTTP method, required input schema, and expected response. This is how you figure out what to send — not by guessing.

**CURIEs are namespaced relations.** Links like `wiki:pages` are CURIEs (Compact URIs). The prefix `wiki:` expands to a documentation URL via a template declared at the API root. `npx hal-walk describe` handles this expansion automatically.

**The session is a graph.** Every `follow` records a position (node) and transition (edge). You can jump back to any previous position with `goto` without re-fetching. The graph can be rendered as a Mermaid diagram or exported as a path spec.
</essential_principles>

<commands>
**Starting and navigating:**

| Command | Purpose |
|---------|---------|
| `npx hal-walk start -s FILE URL` | Begin session, GET the root resource |
| `npx hal-walk position -s FILE` | Show current position, response, and available links |
| `npx hal-walk goto -s FILE POSITION_ID` | Jump to a previous position (local, no HTTP) |

**Discovering and following links:**

| Command | Purpose |
|---------|---------|
| `npx hal-walk describe -s FILE RELATION` | Fetch relation documentation (outputs markdown) |
| `npx hal-walk follow -s FILE RELATION` | Follow a link (GET by default) |
| `npx hal-walk follow -s FILE RELATION --data JSON` | Follow with a request body (POST by default) |
| `npx hal-walk follow -s FILE RELATION --method PUT --data JSON` | Follow with explicit method |
| `npx hal-walk follow -s FILE RELATION --template-vars JSON` | Expand a templated link (RFC 6570) |

**Exporting:**

| Command | Purpose |
|---------|---------|
| `npx hal-walk render -s FILE` | Generate Mermaid diagram of the session graph |
| `npx hal-walk export -s FILE` | Export path from start to current position |
| `npx hal-walk export -s FILE --from POS --to POS` | Export path between specific positions |
</commands>

<exploration_workflow>
**Step 1: Start at the root**

```bash
npx hal-walk start -s session.json http://localhost:3000/
```

Read the output. Look at the `links` array — these are the relations you can follow.

**Step 2: Understand a relation**

Pick a relation from the links. Fetch its documentation:

```bash
npx hal-walk describe -s session.json wiki:create-page
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
npx hal-walk follow -s session.json wiki:pages

# For POST/PUT links (provide --data based on the schema you read):
npx hal-walk follow -s session.json wiki:create-page \
  --data '{"title": "My Page", "body": "Content here", "tags": ["demo"]}'

# For templated links (provide variables):
npx hal-walk follow -s session.json wiki:search \
  --template-vars '{"q": "protocols"}'

# For PUT (explicit method override):
npx hal-walk follow -s session.json wiki:edit-page \
  --method PUT --data '{"body": "Updated content", "editNote": "Fixed typo"}'
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

The path spec is a JSON array of steps with exact relations, methods, and input data. It captures the workflow you discovered so it can be replayed by a simple runner with no inference.

To export a specific sub-path:

```bash
npx hal-walk export -s session.json --from p1 --to p5
```

This uses BFS to find the shortest path between two positions in the session graph.
</distillation>

<method_inference>
The `follow` command defaults to GET. When you provide `--data`, it defaults to POST. For other methods (PUT, DELETE), use `--method` explicitly.

How to determine the correct method:
1. Run `npx hal-walk describe` on the relation
2. Read the "Method" section of the markdown
3. Use `--method` if it's not GET or POST

Common patterns:
- `create-*` relations → POST (inferred from `--data`)
- `edit-*` relations → PUT (use `--method PUT`)
- `delete-*` relations → DELETE (use `--method DELETE`)
- Everything else → GET (default)
</method_inference>

<success_criteria>
Exploration is successful when:
- You navigated the API entirely through link-following (no hardcoded URLs)
- You used `describe` to understand unfamiliar relations before acting
- The session file contains a complete record of your traversal

Distillation is successful when:
- `npx hal-walk render` produces a valid Mermaid graph showing the traversal
- `npx hal-walk export` produces a path spec with concrete steps, relations, and methods
- The path spec could be executed by a runner without any LLM inference
</success_criteria>
