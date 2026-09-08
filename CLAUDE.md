# Pramaan — GeM bid compliance verification

Landing page for **Pramaan**, a prototype built for **Smart India Hackathon 2026, Problem Statement
SIH26100**.

**The product concept:** procurement officers on the Government e-Marketplace (GeM) must verify every
bidder against ~11 disconnected government portals (Udyam, GSTN, PAN/ITD, MCA21, EPFO, ESIC,
DigiLocker, NSIC, Startup India, Make in India/BIS-DPIIT, debarment registers). Pramaan queries them
automatically and produces one scored, auditable compliance record. **The officer still decides** —
this human-in-the-loop framing is central to the positioning and should never be softened into
"the AI approves bidders".

This repo is **frontend only** — a marketing site plus a clickable officer console. No backend,
no real API integration, no working auth. The user will say when to start on the backend.

---

## Stack & commands

Astro 4 static site. **Zero runtime dependencies** — no React/Vue/Tailwind. Plain `.astro`
components, one hand-written global stylesheet, small `is:inline` vanilla scripts.

```bash
npm run dev      # localhost:4321
npm run build    # -> dist/
npm run preview
```

## Deployment — read before touching links

Deployed to GitHub Pages by `.github/workflows/` on push to `main`. The workflow builds with:

```
npx astro build --base /Gem-procurement
```

`astro.config.mjs` deliberately has **`site` but no `base`**, so local dev serves from `/` while
production serves from `/Gem-procurement/`.

**Consequence:** every internal link must go through `import.meta.env.BASE_URL`, never a bare `/`.

```astro
const base = import.meta.env.BASE_URL;   // "/" in dev, "/Gem-procurement/" in prod
<a href={`${base}login`}>          <!-- correct -->
<a href="/login">                  <!-- breaks in production -->
```

`base` always ends in a slash, so concatenate without one. Same-page anchors (`href="#how"`) are fine
bare; cross-page anchors must be `${base}#how` so they work from `/login`.

---

## Structure

```
src/
  layouts/Layout.astro      marketing pages: meta, OG/Twitter, canonical, inline SVG favicon
  layouts/AppLayout.astro   console shell: sidebar + topbar + mobile drawer (owns its own script)
  components/Header.astro   floating pill nav + mobile menu (marketing only)
  components/Footer.astro   4-column footer (marketing only)
  data/mock.ts              every value the console renders — single source of truth
  pages/index.astro         landing page; content lives in frontmatter arrays
  pages/login.astro         demo sign-in; submitting walks through to /app
  pages/app/index.astro     console: overview + decision queue
  pages/app/tender.astro    console: bidder comparison for one tender
  pages/app/bidder.astro    console: the compliance record (the important screen)
  pages/app/portals.astro   console: source integration health
  pages/app/audit.astro     console: full audit trail
  styles/global.css         marketing design system
  styles/app.css            console design system (tokens come from global.css)
public/logos/               real portal logos + SOURCES.md
```

**Two surfaces, one token set.** Marketing pages import `global.css`; console pages import
`global.css` (for `:root` tokens and fonts) *then* `app.css`. `app.css` adds `--alert`, `--info`,
`--side-w`, `--top-h` and owns everything from `.app` downwards.

Legacy at repo root, **not part of the build**: `pramaan-landing.html` (the original single-file
version) and `convert.js`. Safe to ignore; delete only if asked.

### Page order (deliberately short — it is a landing page)

**Hero → portals marquee → How it works → FAQ → closing CTA → footer.**

Only `#how` and `#faq` are nav targets. Earlier revisions also had Problem, Capabilities, Impact,
a Tender board and an Evidence chain section; **the user removed all of them on purpose.** Don't
reintroduce sections uninvited — they're in git history if ever wanted back.

Content is **data-driven from frontmatter arrays** (`portals`, `steps`, `faqs`). Edit the array, not
the markup.

---

## Design system (`src/styles/global.css`)

Civic/editorial "ledger" aesthetic — deliberately **not** a generic purple-gradient SaaS page. Warm
paper ground, deep navy, gold seal accent, green for verified.

| Token group | Notes |
|---|---|
| Ground | `--paper` page, `--paper-raised` alternating sections, `--surface` white cards |
| Brand | `--navy` / `--navy-deep`, `--verify` green, `--seal` gold, `--flag` amber for review states |
| Elevation | `--shadow-xs` → `--shadow-xl`; cards use `xs`, hover `lg`, the hero mockup `xl` |
| Radii | `--r-xs` → `--r-xl`, `--r-pill` |
| Type | `--serif` Fraunces (headings), `--sans` IBM Plex Sans (body), `--mono` IBM Plex Mono (labels, IDs, timestamps) |

**Rules that keep it coherent:**

- Mono is for *machine* text only — timestamps, record IDs, portal names, eyebrows, status pills.
  Never body copy.
- Grounds alternate: hero `paper` → portals `raised` → how `paper` → FAQ `raised` → closing `paper`.
  Adding or removing a section means re-checking this alternation; two adjacent same-ground sections
  look like a mistake.
- Section separation is a single `border-top`. `.portals` already has a `border-bottom`, so the
  section after it (`.steps`) carries `border-top:none` — don't "fix" that into a doubled 2px line.
- Green = verified, amber = needs officer review, gold = accent/seal. Amber is never an error colour;
  **nothing on this site is red.**

### The officer console (`/app`)

Frontend only — **no backend yet, by explicit instruction.** Forms and buttons are inert except the
login submit, which navigates to `/app`.

Routes: `/app` (overview + queue) -> `/app/tender` (bidder comparison) -> `/app/bidder` (compliance
record) plus `/app/portals` and `/app/audit`. `AppLayout` takes `title`, `active` (sidebar highlight)
and `crumbs`.

Everything renders from `src/data/mock.ts`, so the same bidder tells one consistent story across
screens. The focus bidder is **Nova Labtech (78/100, Medium)** — deliberately neither a clean pass
nor an obvious reject, so the record has to show the engine reasoning and hand a real judgement over.

Rules this console is built on, worth preserving:

- **Every value carries provenance.** A check row shows source portal, submitted value, retrieved
  value, verdict and timestamp. Never render a bare status.
- **Score is never the decision.** A bidder can score 78 and still fail one qualifying clause. The
  ranking is a review convenience; the decision panel is the point of the screen.
- **A missing source is pending, not passing.** `/app/portals` exists to make that visible.
- The AI recommendation card carries a **confidence figure and an explicit limits note**. Keep both
  if you touch it.
- `.tbl` rows set `grid-template-columns` inline per page and carry `min-width:680px`; the enclosing
  `.panel-body.flush` scrolls horizontally so the page body never does.
- The score ring animates via `--off` (a stroke-dashoffset computed in the page, circumference
  282.7 at r=45) and is disabled under `prefers-reduced-motion`.

### The nav is a floating pill

`header.nav` is `position:sticky; top:16px` and `.nav-inner` is fully rounded (`--r-pill`) with a
translucent blurred background. It gets `.scrolled` from JS past 8px to deepen the shadow. There is
**no announcement bar above it** — one existed and was removed; don't add it back.

### The hero mockup carries the page

`.app-shell` — browser chrome plus a bidder compliance dashboard: animated 96/100 gauge, six
staggered verification rows (five matched, one flagged amber), floating "42 min saved" stat card,
subtly rotated in 3D and straightening on hover. With the page this short, it is doing most of the
"this is a real product" work, so treat it as load-bearing.

### Portal logos are real, with two drawn fallbacks

The marquee uses **actual government portal logos** in `public/logos/` — see
[`public/logos/SOURCES.md`](public/logos/SOURCES.md) for the per-file source and licence table.
Each `portals` entry in `index.astro` is either:

- `{ n, logo: 'epfo.png' }` -> renders `<img src={`${base}logos/...`}>`, or
- `{ n, c, icon }` -> renders the hand-drawn SVG glyph (only **GSTN** and **debarment**, which have
  no obtainable/undefined logo).

Rasters are trimmed + downscaled to a 128-160px box and shown in a fixed **54x26** `.chip-logo` box
with `object-fit:contain`, so square emblems and wide wordmarks share one optical footprint without
distorting. Logo `src` **must** go through `${base}` like every other asset.

Two constraints worth keeping:

- `mca21.svg` is **CC BY-SA 4.0** (attribution + share-alike); `pan.png` is public domain. The rest
  are official marks used nominatively to identify the portal being integrated with.
- **Never use the bare State Emblem of India.** A crop of the MSME lockup once reduced it to exactly
  that and was reverted — its use is restricted by the State Emblem of India (Prohibition of Improper
  Use) Act, 2005, and it identifies no particular portal.

### Motion

`.reveal` + `IntersectionObserver` for scroll-in; a count-up on the diagram score.
`prefers-reduced-motion` is honoured globally *and* with explicit per-component overrides (gauge fill,
app rows, float card) — add an override whenever you add an animation that starts at `opacity:0`, or
reduced-motion users get invisible content.

---

## Conventions

- Scripts are `is:inline` vanilla JS in an IIFE, guarded with null checks and an
  `IntersectionObserver in window` fallback. Header owns its own script; page behaviour lives at the
  bottom of `index.astro`.
- Nav breakpoint is **1000px** in both `global.css` and the resize handler in `Header.astro`. Change
  both together.
- Decorative SVGs get `aria-hidden="true"`; meaningful ones get `role="img"` + `aria-label`.
- `.audit-point*` in the FEATURE POINTS block is used **only by `login.astro`**. It survived a
  section deletion once already — check `login.astro` before removing it.

---

## Content caveat — matters for judging

Several figures are **illustrative, not measured**: "42 min saved per bidder" and every value in the
hero mockup (bidder name, tender ID, scores, timings). Don't add *new* invented metrics — fabricated
statistics are the fastest way to lose credibility in a procurement context. The login page is
labelled "Demo environment · credentials are not transmitted" because the form does nothing; keep a
note like that as long as that stays true.

---

## Gotchas

- **An element inside a grid/flex container must have base styles, not only styles inside a
  `@media` block.** `.scrim` (the console's mobile drawer overlay) is a child of `.app` and was
  originally given `position:fixed` only under `max-width:960px`. At desktop widths it fell back to a
  plain block, took grid **column 2**, and pushed `.main` onto the next row — every console page
  rendered a sidebar next to blank space. It now carries `display:none` at the top level and
  `display:block` inside the query. Same trap applies to anything else dropped into `.app`.
- **Large bash heredocs to write files have failed in this environment** (`unexpected EOF while
  looking for matching`) once the chunk got long, even with a quoted delimiter. Use the Write tool,
  or a Python heredoc, for anything sizeable.
- After editing `global.css`, sanity-check `{` vs `}` counts — Astro/Vite will happily bundle
  unbalanced CSS and the breakage only shows up visually.
- After deleting a section, grep for orphaned anchors (`#capabilities`, `#impact`, …) in
  `Header.astro`, `Footer.astro` and `login.astro`, and for CSS classes with no markup left.
- No browser or screenshot tooling is available here. `npm run build` plus grepping `dist/` for
  expected classes is the practical verification; real visual checks need the user.
