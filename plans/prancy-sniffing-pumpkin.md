# Deploy agent-wiki to Cloudflare Workers with D1

## Context

The agent-wiki Express server uses an in-memory store and filesystem reads for rels docs. To deploy to Cloudflare, we need to swap Express for Hono, replace the in-memory Maps with D1 (SQLite), and embed the markdown rels as strings. The HAL response shapes, routes, and behavior stay identical.

## Approach

Create a new `agent-wiki-workers/` directory (sibling to `agent-wiki/`) rather than modifying the original. This keeps the simple Express version as a local dev reference.

## D1 Schema

```sql
CREATE TABLE pages (
  id              INTEGER PRIMARY KEY AUTOINCREMENT,
  title           TEXT NOT NULL,
  body            TEXT NOT NULL,
  tags            TEXT NOT NULL DEFAULT '[]',
  created_at      TEXT NOT NULL,
  updated_at      TEXT NOT NULL,
  current_version INTEGER NOT NULL DEFAULT 1
);

CREATE TABLE versions (
  vid        INTEGER NOT NULL,
  page_id    INTEGER NOT NULL REFERENCES pages(id) ON DELETE CASCADE,
  title      TEXT NOT NULL,
  body       TEXT NOT NULL,
  tags       TEXT NOT NULL DEFAULT '[]',
  edit_note  TEXT,
  diff       TEXT,
  created_at TEXT NOT NULL,
  PRIMARY KEY (page_id, vid)
);

CREATE TABLE comments (
  id         TEXT PRIMARY KEY,
  page_id    INTEGER NOT NULL REFERENCES pages(id) ON DELETE CASCADE,
  vid        INTEGER NOT NULL,
  author     TEXT NOT NULL,
  body       TEXT NOT NULL,
  created_at TEXT NOT NULL,
  FOREIGN KEY (page_id, vid) REFERENCES versions(page_id, vid) ON DELETE CASCADE
);

CREATE INDEX idx_versions_page_id ON versions(page_id);
CREATE INDEX idx_comments_page_vid ON comments(page_id, vid);
```

Categories are derived at query time via `json_each(tags)` — no table needed.

## File Structure

```
agent-wiki-workers/
  wrangler.toml
  package.json
  tsconfig.json
  migrations/
    0001_initial.sql
  src/
    index.ts          — Hono app + hal+json middleware
    types.ts          — Copied verbatim from original
    hal.ts            — Same helpers, BASE_URL from env instead of process.env
    store.ts          — Complete rewrite: async D1 queries
    rels.ts           — Markdown content embedded as Map<string, string>
    routes/
      root.ts
      pages.ts
      versions.ts
      comments.ts
      categories.ts
      search.ts
      rels.ts
```

## Key Changes

| Component | Express version | Workers version |
|-----------|----------------|-----------------|
| Framework | Express Router | Hono |
| Storage | In-memory Maps | D1 SQLite |
| Rels docs | `readFileSync` from disk | Embedded string Map |
| UUIDs | `uuid` package | `crypto.randomUUID()` |
| BASE_URL | `process.env` | `c.env.BASE_URL` (wrangler.toml `[vars]`) |
| Diffs | `diff` package | `diff` package (pure JS, works in Workers) |

**What's unchanged:** `types.ts` (verbatim), all HAL helper functions in `hal.ts` (logic identical, just parameterize `baseUrl`), all route business logic (mechanical Express→Hono translation).

## Express → Hono Translation Pattern

| Express | Hono |
|---------|------|
| `req.params.id` | `c.req.param('id')` |
| `req.body` | `await c.req.json()` |
| `req.query.q` | `c.req.query('q')` |
| `res.json(data)` | `return c.json(data)` |
| `res.status(404).json(data)` | `return c.json(data, 404)` |
| `res.status(204).send()` | `return c.body(null, 204)` |

All handlers become `async` since D1 queries are async.

## Store Migration Summary

Every store function keeps the same signature (plus a `db: D1Database` first param) and return type. Implementation changes from Map ops to SQL:

- `createPage` → INSERT into pages + versions, return from `last_row_id`
- `editPage` → Read prev version, compute diff, batch INSERT version + UPDATE page
- `deletePage` → DELETE page (CASCADE handles versions + comments)
- `getPage/getAllPages` → SELECT with `rowToPage()` helper (parses JSON tags)
- `getVersions/getVersion` → SELECT from versions
- `addComment/getComments/getComment` → INSERT/SELECT from comments, `crypto.randomUUID()`
- `getAllCategories` → `SELECT tag.value, GROUP_CONCAT(p.id) FROM pages p, json_each(p.tags) tag GROUP BY tag.value`
- `getCategory` → `SELECT p.id FROM pages p, json_each(p.tags) tag WHERE tag.value = ?`
- `searchPages` → `SELECT WHERE title LIKE ? OR body LIKE ? OR tag.value LIKE ?` with COLLATE NOCASE

## Implementation Order

1. Scaffold project (package.json, tsconfig, wrangler.toml, npm install)
2. D1 migration file + apply locally
3. Copy `types.ts`, adapt `hal.ts` (parameterize baseUrl)
4. Build `store.ts` (D1 queries — largest piece of work)
5. Build `rels.ts` (embed markdown strings)
6. Migrate all route files (Express → Hono, sync → async)
7. Wire up `index.ts` (Hono app, middleware, route mounting)
8. Test locally with `wrangler dev`

## Verification

1. `wrangler d1 migrations apply agent-wiki-db --local`
2. `wrangler dev` — run full smoke test against every endpoint:
   - GET / (root with curies)
   - POST /pages, GET /pages, GET /pages/:id
   - PUT /pages/:id (verify version increment + diff)
   - GET /pages/:id/history, GET /pages/:id/versions/:vid
   - POST + GET comments
   - GET /categories, GET /categories/:name
   - GET /search?q=...
   - DELETE /pages/:id (verify cascade)
   - GET /rels/:name (verify text/markdown content type)
3. Verify HAL response shapes match the Express version exactly
4. Deploy with `wrangler deploy` + `wrangler d1 migrations apply --remote`
