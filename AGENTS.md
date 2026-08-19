# AGENTS.md — content (Toscana Wine Bar CMS backend)

## What this repo is

A **git-based CMS backend** for the Toscana Wine Bar website, built on
[Decap CMS](https://decapcms.org/) (formerly Netlify CMS).

- Editors log in at `/admin/` and edit structured content through a form UI.
- Decap CMS commits the changes as JSON/image files **directly to this git repo**
  via the GitHub API — there is no database.
- GitHub repo: `WineBarProstejov/content`, branch `main`.
- Deployed as a static site on **Cloudflare Pages**: `https://content-87t.pages.dev`.
- OAuth for the GitHub backend is handled by Cloudflare Pages Functions in
  `functions/api/auth.js` and `functions/api/callback.js`.

The sibling repo **[`website`](../website/AGENTS.md)** (Next.js frontend, deployed on
Vercel) reads the JSON files this repo publishes, over plain HTTP, at runtime.
**This repo and `website` are one system with two deploy targets.** Nothing here
has meaning on its own — every field defined below exists because a specific
component in `website` reads it by that exact path and shape.

## Non-negotiable rule

> **Never rename, remove, or reshape a collection, file path, or field in
> `admin/config.yml` without making the matching change in `website` in the
> same change set — and vice versa.**

There is no type checking or schema validation across the two repos. A rename
here fails **silently** in the frontend (React state just stays empty/`undefined`
with the `try/catch` swallowing the fetch mismatch) — nothing errors loudly, so
it's easy to ship a broken site by accident. Before touching `admin/config.yml`
or any published JSON path, grep `website/app` for the current path/field name
first.

## The content contract

All published content lives under `public/` (this is both the Decap
`media_folder`/`public_folder` root *and* the path the frontend fetches from).

| Collection (`admin/config.yml`) | Published file | Consumed by (in `website`) |
|---|---|---|
| `welcome` | `public/content/app/components/welcome.json` | `app/components/Welcome.tsx` (hero subtitle) |
| `rating` | `public/content/app/components/rating.json` | `app/components/GoogleRatingBadge.tsx` (hero) |
| `events` | `public/content/app/components/events.json` | `app/components/Events.tsx` (homepage) |
| `about` | `public/content/app/components/about.json` | `app/components/About.tsx` |
| `galerie` | `public/content/app/components/galerie.json` | `app/components/Gallery.tsx` |
| `oteviracka` | `public/content/app/components/oteviracka.json` | `app/components/Oteviracka.tsx`, `app/components/open.tsx` |
| `menu` | `public/content/app/components/menu.json` | `app/lib/content.ts`'s `getMenu()` |

Uploaded images live under `public/uploads/` (top-level media) or under
collection-specific subfolders such as `public/content/app/components/interier/`,
`.../galerie/`, `.../galerie/menu/` (see each collection's `media_folder` /
`public_folder` in `admin/config.yml`). Image paths inside the JSON are stored
**relative** (e.g. `public/content/app/components/galerie/5.webp`) and the
frontend resolves them against its configured content base URL — do not switch
these to absolute URLs pointing at a different host.

### Field shapes (as actually consumed today)

- **welcome.json**: `subtitle` (string) — the one-line tagline under "Toscana
  Wine Bar" on the homepage hero.
- **events.json**: `items[]` (`title`, `date` ISO `YYYY-MM-DD`, `description`,
  `image?`). `app/lib/content.ts`'s `getEvents()` filters out events more
  than a day in the past and sorts by date — the CMS list order doesn't
  matter. An empty `items[]` is valid; `Events.tsx` renders nothing rather
  than an empty "Připravujeme" heading.
- **rating.json**: `rating` (number, e.g. `4.9`), `count` (optional string,
  e.g. `"126"`), `url` (optional link to the business's Google reviews).
  Shows as a "4.9★ na Googlu" badge in the hero (`GoogleRatingBadge.tsx`) —
  deliberately just the aggregate score, not individual reviews. This is
  **manually typed in, not fetched live from Google** — pulling it live
  would need a Google Cloud project with a billing account attached (Places
  API), which was rejected during the redesign pass as unnecessary
  cost/setup for a number that changes rarely. The site owner updates
  `rating` by hand in the CMS when the real Google rating changes. Never
  invent/bump this number without the owner telling you the real current
  value.
- **about.json**: `title` (string), `description` (text), `timeline[]` (`year`,
  `description`, `image`, `alt`), `vision.title`, `vision.description`,
  `vision.mission.{title,description}`, `vision.values.{title,list[],feedback}`.
- **galerie.json**: `photos[]` (`src`, `alt`).
- **oteviracka.json**: `title`, `day1`..`day7` (`title`, `hours` — either a
  free-text string like `"16:00 - 22:00"` or `"Zavřeno"`). `open.tsx`'s live
  "Otevřeno/Zavřeno" badge parses `day1..day7.hours` directly for today's
  weekday — this is the single source of truth for whether the bar is open.
  There is also a legacy `funkce.{poctv,patek,sobota}.{od,do}` field
  (separate Po-Čt/Pátek/Sobota hour ranges) that **duplicates** the same
  information and can drift out of sync with `day1..day7` (it did — `poctv`
  claimed Monday was open 16–22 while `day1.hours` said `"Zavřeno"`). Nothing
  in `website` reads `funkce` anymore; don't reintroduce a second reader of
  it, and prefer editing `day1..day7` as the one real source of the hours.
- **menu.json**: `cover.{image,alt}`, `images[]` (`src`, `alt`).

CORS is opened for `/content/*` and `/uploads/*` in `_headers` (read by
Cloudflare Pages) — this is required for `website` to fetch cross-origin.
`Cache-Control: no-store` on `/content/*` is intentional so editors see fresh
content immediately; don't add caching there without checking with `website`'s
revalidation strategy first.

## Known quirks (not urgent, but don't be surprised)

- **`netlify.toml`** at repo root configures Netlify-style headers/CORS, but
  this site is deployed on **Cloudflare Pages**, which reads `_headers`
  instead. `netlify.toml` is very likely a dead leftover from before the host
  migration — `_headers` is the file that actually takes effect. Confirm before
  deleting.
- **`dev/config.yml`** is a separate Decap config using `local_backend: true`
  + `git-gateway`, for running the CMS against a local dev server instead of
  the GitHub backend. It duplicates the collections in `admin/config.yml` — if
  you change one, check whether the other needs the same change.
- **`content/content/`** (note: `content/content/app/components/about.json`)
  contains a divergent draft copy of `about.json` that differs from the
  published one under `content/public/content/app/components/about.json`.
  Only the file under `public/` is what `website` actually fetches; the other
  looks like a stale working copy.
- **`content/content/posts/`** (`new.json`, `asd.json`) has no matching
  `posts` collection in `admin/config.yml` and nothing in `website` reads it —
  looks like an abandoned experiment, not part of the live contract.

## Making changes safely

1. Read this file and `website/AGENTS.md` before editing `admin/config.yml`.
2. If you add/rename a field or file path, update the consuming component in
   `website` in the same change, and mention it in your commit message.
3. Test by publishing through `/admin/` (or `dev/config.yml` locally) and
   confirming the resulting JSON still matches what `website` expects.
4. Don't introduce new top-level content types without also deciding where in
   `website` they'll render — an orphaned collection is exactly how
   `content/content/posts/` happened.
