# Hermes Output Portal — Spec

## Problem

The Hermes CTO loop now produces many single-page HTML artefacts:
playbooks, meeting summaries, articles, briefs, post-mortems, status reports,
and more types as new skills land. They are scattered — chat messages, the
VPS filesystem, the founder's Downloads folder. There is no single place to
find them, no history, no search, no shareable home.

The portal is one site that lists every HTML output Hermes has ever
produced, grouped by type, searchable, and viewable in-browser.

## Goals

1. **One URL.** Founder bookmarks one address; everything Hermes has ever
   produced is one click away.
2. **Zero-friction publish.** A skill takes an HTML file and a few fields,
   commits it, and the portal updates automatically. No CMS, no upload UI.
3. **Versioned.** Every output is a file in git. History, diff, and
   restore for free.
4. **Type-aware.** The portal has tabs per type (playbook, meeting summary,
   article, …) so adding a new output type is a one-line config change.
5. **Private to the founder.** Meeting summaries and internal playbooks
   never get a Google-indexable URL.

## Non-goals

- A general-purpose CMS. Editing happens in the source skill, not the portal.
- A multi-tenant SaaS. One founder, one portal.
- Server-side rendering. Static-only.
- Rich-text WYSIWYG. The skill outputs final HTML.

---

## Architecture

```
┌────────────────────────────┐         ┌──────────────────────────┐
│ Hermes (VPS)               │         │ GitHub: hermes-portal    │
│                            │         │ (private repo)           │
│  skill: publish-to-portal  │ commit  │                          │
│   ├─ writes HTML file      ├────────►│ /site/         (shell)   │
│   ├─ regen manifest.json   │  push   │ /content/<type>/<slug>   │
│   └─ git push              │         │ /manifest.json           │
└────────────────────────────┘         └────────────┬─────────────┘
                                                    │ GitHub Action
                                                    │ deploys /site
                                                    ▼
                                       ┌──────────────────────────┐
                                       │ GitHub Pages             │
                                       │ portal.<domain>          │
                                       │ static shell + JS auth   │
                                       └────────────┬─────────────┘
                                                    │ magic link
                                                    ▼
                                       ┌──────────────────────────┐
                                       │ Supabase                 │
                                       │ - auth (magic link)      │
                                       │ - edge fn: get-doc       │
                                       │   (holds GH PAT, returns │
                                       │   private file content)  │
                                       └──────────────────────────┘
```

### Why this shape

- **Static Pages**: cheapest, simplest, no runtime to babysit.
- **Single private repo**: keeps content and shell together so the
  `publish-to-portal` skill has one git remote to push to.
- **Supabase magic link**: matches the rest of the Oh My Hermes stack.
- **Edge function with GH PAT**: GitHub Pages cannot keep HTML files
  truly private on its own. The shell is public-by-URL but useless without
  auth; the actual HTML bodies are fetched through a Supabase Edge Function
  that calls the GitHub Contents API with a server-side PAT and gates on
  the caller's Supabase JWT. The static URLs of the HTML files themselves
  don't resolve directly.

### Sharp edge: GitHub Pages + private repo

GitHub Pages serving from a **private repo requires GitHub Pro / Team /
Enterprise** on the repo owner's account. Three options if the user is on
free:

1. Pay for Pro on the repo owner account (cheapest, $4/mo).
2. Make `/site` a public sub-path of an otherwise-public repo and only
   keep `/content` private — split into two repos. Adds wiring.
3. Switch hosting to Vercel (supports auth-gating + private deploys
   natively). Easiest if Pro isn't an option.

**Default recommendation:** option 1. Fall back to option 3 if the founder
declines to upgrade.

---

## Repository layout (`hermes-portal`)

```
hermes-portal/
├── site/                       # static shell, deployed to Pages
│   ├── index.html              # tabs, list view, auth gate
│   ├── view.html               # iframe sandbox renderer
│   ├── app.js                  # supabase client, manifest fetch, routing
│   ├── styles.css
│   └── config.json             # supabase URL + anon key, doc types
│
├── content/
│   ├── playbook/
│   │   └── 2026/05/<slug>.html
│   ├── meeting-summary/
│   │   └── 2026/05/<slug>.html
│   ├── article/
│   ├── status-report/
│   └── post-mortem/
│
├── manifest.json               # generated; the index the UI reads
├── .github/workflows/
│   └── deploy-pages.yml
└── README.md
```

### `manifest.json` shape

```json
{
  "version": 1,
  "generated_at": "2026-05-13T09:00:00Z",
  "types": ["playbook", "meeting-summary", "article", "status-report", "post-mortem"],
  "docs": [
    {
      "id": "2026-05-12-q2-launch-playbook",
      "type": "playbook",
      "title": "Q2 Launch Playbook",
      "summary": "Sequencing for the May launch — owners, dates, kill criteria.",
      "path": "content/playbook/2026/05/q2-launch-playbook.html",
      "tags": ["launch", "q2"],
      "created_at": "2026-05-12T14:33:00Z",
      "size_bytes": 18402,
      "source": { "skill": "playbook-author", "run_id": "abc123" }
    }
  ]
}
```

The manifest is the only thing the UI loads on boot. Adding a new doc =
appending one entry and rewriting the file. Listing 10k docs stays fast.

---

## The `publish-to-portal` skill

New skill in `oh-my-hermes/skills/publish-to-portal.md`. Signature:

```yaml
inputs:
  html_path: string           # absolute path to the HTML file Hermes just produced
  type: enum                  # playbook | meeting-summary | article | status-report | post-mortem | ...
  title: string
  summary: string             # 1–2 sentence preview shown in the list
  tags: string[]              # optional
  source:                     # optional, for traceability
    skill: string
    run_id: string
```

### Steps

1. Resolve the portal repo path from memory key `portal-repo-path`
   (default `~/.hermes/portal`). If absent, clone from
   `portal-repo-remote` memory key.
2. Slugify `title`. Build target path
   `content/<type>/YYYY/MM/<slug>.html`. If it exists, append `-2`, `-3`, …
3. Copy `html_path` to target. Run a quick HTML safety pass:
   - strip `<script src="http">` to local-only,
   - reject `<iframe>` to non-allowlisted hosts,
   - wrap body in a `<main>` if missing (so the renderer's CSS resets cleanly).
4. Read `manifest.json`, append the new entry, sort by `created_at` desc,
   write back.
5. `git add . && git commit -m "publish: <type> — <title>" && git push`.
6. Return the canonical view URL: `https://portal.<domain>/view?id=<id>`.

### Pitfalls (only fill in once observed)

_(empty until first real run)_

---

## Portal UI

### Auth gate (`index.html` + `app.js`)

- Boot → check Supabase session → if none, show magic-link form.
- Allowed emails come from a Supabase table `portal_allowlist` with RLS
  permitting `select` only on `auth.email() = email`. If the user's email
  isn't in the row they get, sign-out + "not invited" message.
- On success, fetch `manifest.json` directly from the repo via the Edge
  Function (so we don't have to make the manifest public either).

### List view

- Top: search box (client-side fuzzy over title/summary/tags).
- Left rail: type tabs — counts per type, "All" at top.
- Right: list of cards sorted by `created_at` desc:
  `title · type · YYYY-MM-DD · summary · tags`.
- Click a card → `/view?id=<id>`.

### Doc view (`view.html`)

- Calls Edge Function `get-doc(id, jwt)` → returns raw HTML.
- Renders in a sandboxed `<iframe sandbox="allow-same-origin">` so doc
  styles can't leak into the shell and doc scripts can't read the JWT.
- Top bar: back, copy-link (deep link with `id`), download, "open raw"
  (also auth-gated).

### Supabase Edge Function `get-doc`

```ts
// pseudo
export default async (req) => {
  const jwt = req.headers.get("Authorization")?.replace("Bearer ", "");
  const user = await verifySupabaseJwt(jwt);          // 401 otherwise
  if (!(await isAllowed(user.email))) return resp(403);

  const id = new URL(req.url).searchParams.get("id");
  const path = await lookupPath(id);                  // from manifest cache
  const html = await githubContents(REPO, path, GH_PAT);
  return new Response(html, { headers: { "content-type": "text/html" } });
};
```

Secrets: `GH_PAT` (read-only, scoped to one repo), Supabase service role
(for allowlist check), repo coordinates. Stored as Edge Function env vars.

---

## Doc types (initial)

| Type             | Source skill (existing or planned) | Notes                       |
|------------------|-------------------------------------|-----------------------------|
| `playbook`       | _planned: `playbook-author`_        | Operational runbooks        |
| `meeting-summary`| _planned: `meeting-summary`_        | Transcript → HTML summary   |
| `article`        | _planned: `article-writer`_         | Long-form writing           |
| `status-report`  | `cto-status-report`                 | Daily founder report        |
| `post-mortem`    | _planned_                           | Incident write-ups          |

Adding a new type = one entry in `site/config.json` and a directory under
`content/`. No code change.

---

## Rollout

1. **Day 0** — Create `hermes-portal` repo (private). Drop in `site/`
   shell + an empty `manifest.json` + Pages workflow. Verify it deploys
   and the auth gate works with one allowlisted email.
2. **Day 1** — Author `publish-to-portal` skill. Publish a hand-rolled
   sample HTML doc per type to seed the UI.
3. **Day 2** — Wire `cto-status-report` to call `publish-to-portal`
   after generating its HTML so the daily report ends up in the portal.
4. **Day 3+** — Each new HTML-producing skill ends its run with
   `publish-to-portal`. New types added to `site/config.json` as they appear.

## Open questions

- **Domain.** `portal.<founder-domain>` vs. the default
  `<owner>.github.io/hermes-portal`. Custom domain is one DNS record;
  punt to founder.
- **Mobile.** Phones will be the primary read surface for status reports
  in the morning. Shell needs to be responsive day 1; doc HTML quality is
  the skill author's problem.
- **Retention.** Do we keep everything forever, or auto-archive
  `meeting-summary` older than N months? Default to forever — git makes
  it cheap.
- **Search depth.** Client-side over title/summary/tags is enough for
  ~1k docs. Past that, generate a Lunr index at publish time.
- **Sharing.** Out of scope for v1 — everything is founder-only. If we
  need to share a single doc externally later, add a per-doc "publish
  publicly" toggle that copies the file to a public bucket.
