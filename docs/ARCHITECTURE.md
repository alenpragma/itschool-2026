# Architecture — Edulife IT School

How the layers fit together, and why they are arranged this way. Read this before changing
`scripts/build.js`; everything else in the project follows from the contract described here.

---

## 1. The stack, stated plainly

**Plain HTML + CSS + vanilla JavaScript, assembled by a custom Node static-site generator.**

- No React. No Next.js. No Vue. No bundler, no transpiler, no CSS preprocessor.
- No runtime dependency. `scripts/build.js` uses only `fs`, `path` and `crypto`.
- `npm install` is **not** required to build, preview or run the quality gate. It is only
  needed for `npm run lint` and `npm run format`.

The only tool is Node (>= 18, per `package.json > engines`; `.nvmrc` pins the tested version).

---

## 2. The four layers

```
  src/data/*.json          CONTENT        the only layer a non-developer edits
        |
  src/components/*.html    PARTIALS       header, drawer, footer, lightbox
  src/layouts/base.html    SHELL          <head>, slots, script tags
  src/pages/*.html         BODIES         tokens + {{BLOCK_*}} placeholders
        |
  scripts/build.js         GENERATOR      reads the above, writes dist/
        |
  dist/                    OUTPUT         the publishable site
```

The build is **one-way and deterministic**: same inputs, same bytes out. Nothing in `dist/`
is written by hand, and nothing in `dist/` feeds back into a later build.

---

## 3. How one page is assembled

For each entry in the `PAGES` array in `scripts/build.js`:

1. **Read the body** — `src/pages/<src>`.
2. **Build the shared tokens** via `baseTokens(page.key)`. This is what makes the header
   show `class="active"` on the right nav item: the page's `key` is matched against
   `site.nav[].key`.
3. **Expand blocks** — every `{{BLOCK_*}}` the body references is called and its HTML
   substituted from the JSON via a renderer.
4. **Substitute tokens** — `{{TOKEN}}` replacement is a plain global string split/join, not
   a regex, so a value containing `$&` or backslashes cannot corrupt the output.
5. **Strip leftovers** — any `{{TOKEN}}` a page did not use is removed and recorded as a
   build warning, so a stray placeholder can never reach the browser.
6. **Wrap in the layout** — `src/layouts/base.html` with `{{CONTENT}}` replaced.
7. **Stamp assets** — every `assets/css/*.css` and `assets/js/*.js` reference gains
   `?v=<stamp>`.
8. **Write** to `dist/<out>`.

### The cache-busting stamp

```js
function assetVersion() {
  const h = crypto.createHash('sha1');
  LOCAL_ASSETS.forEach(f => h.update(fs.readFileSync(path.join(SRC, f))));
  return h.digest('hex').slice(0, 8);
}
```

One SHA-1 over the concatenated bytes of every local CSS/JS file, truncated to 8 hex chars.
Every page in a build shares the stamp. **Consequence:** editing any CSS or JS file changes
the stamp on every page, which is what makes `immutable` caching on `/assets/*` safe — a
changed file always gets a new URL. Editing only `src/data/*.json` or a page body leaves the
stamp alone, because no asset byte changed.

### Detail pages

`src/pages/detail.html` is **one template for all three families** (blog, notice, event). It
carries a single token, `{{DETAIL_CONTENT}}`, filled by `R.renderDetail(type, item, {...})`.

Because detail pages live one directory down (`blog/<slug>.html`), a root-relative href such
as `assets/css/styles.css` would resolve against `blog/` and 404. Rather than maintain a
second set of templates, each generated detail page is run through `rebase(html, 1)`, which
prefixes relative URLs with the correct number of `../` hops. Absolute URLs (`http:`,
`mailto:`, `tel:`, `data:`), protocol-relative (`//`), in-page anchors (`#`) and
already-relative paths (`./`, `../`) are all left untouched.

This is why the project needs **no `<base href>`** and works equally at a domain root and at
a sub-path such as `https://user.github.io/repo/`.

### Why comments are stripped before substitution

Templates carry developer documentation in HTML comments, and some of those comments name
the very tokens being substituted. If substitution ran first, the placeholder would expand
*inside* the comment, breaking it and emitting the component twice. `stripComments()` runs
before `apply()` for exactly that reason.

---

## 4. Why source and output are separate directories

`dist/` is **gitignored and wiped at the start of every build**:

```js
fs.rmSync(OUT_DIR, { recursive: true, force: true });
fs.mkdirSync(OUT_DIR, { recursive: true });
```

That clean step is not cosmetic. Without it, renaming or deleting a page leaves the old file
in place, and it keeps resolving on the host forever — a 404 that never fires and a sitemap
that advertises a page nobody maintains.

The split also means a reviewer reads `src/`, and CI validates generated output, instead of
both parties staring at 31 files of generated markup.

---

## 5. The data layer and the REST seam

`src/data/*.json` is the content layer. `build.js` reads it at build time; the browser can
*also* read it at runtime, which is the migration path to a backend.

| Layer | File | Runtime location |
|---|---|---|
| Site / global | `src/data/site.json` | `dist/data/site.json` |
| Teachers | `src/data/teachers.json` | `dist/data/teachers.json` |
| Notices | `src/data/notices.json` | `dist/data/notices.json` |
| Events | `src/data/events.json` | `dist/data/events.json` |
| Gallery | `src/data/gallery.json` | `dist/data/gallery.json` |
| Blog | `src/data/blog.json` | `dist/data/blog.json` |

The build copies `src/data/` into `dist/data/` deliberately. `src/assets/js/data-loader.js`
holds the seam: set `apiBase` and flip `<body data-live="true">`, and the same renderers that
produced the static HTML now fetch from your API instead. Because `build.js` already wrote
real cards into the HTML, the pages are fully readable and crawlable **before** that switch is
thrown — the runtime fetch is an enhancement, never a prerequisite.

---

## 6. Renderers: shared by Node and the browser

`src/assets/js/renderers.js` is written as a UMD-style module so the *same* functions produce
markup in both environments:

- `build.js` does `require('src/assets/js/renderers.js')` and calls `renderList`,
  `renderDetail`, `esc`.
- The browser loads it with a plain `<script>` tag and uses the identical functions for the
  data-loader path.

One implementation means the static output and the runtime output cannot drift apart. Every
value passes through `esc()`, so text from a JSON file can never inject markup.

---

## 7. The quality gate

`scripts/check.js` runs structural assertions over `dist/` (generated output only — scanning
templates would produce hundreds of false dead links):

- HTML tag balance across every generated page
- no unresolved `{{TOKEN}}` anywhere in `dist/`
- the full link graph resolves (no dead internal links or asset paths)
- nav/class-dropdown data stays in sync
- no orphan CSS rules
- logotype and initiative-logo contracts (file exists, and is genuinely PNG/JPEG bytes)

The GitHub Pages workflow runs it **before** deploying, so a build that fails the gate never
reaches production. `npm run verify` is the same pair of steps locally.

---

## 8. Build, preview, deploy

| Command | Does |
|---|---|
| `npm run build` | `node scripts/build.js` → rebuilds `dist/` |
| `npm run check` | `node scripts/check.js` → quality gate |
| `npm run verify` | build, then check |
| `npm run dev` | `node scripts/serve.js` → serves `dist/` on localhost:3000 |
| `npm run clean` | deletes `dist/` |

`scripts/serve.js` is a zero-dependency preview server: correct MIME types, directory index,
clean-URL resolution and a `404.html` fallback, so local behaviour matches a real host.

All three hosting configs publish **`dist/`**:

- `.github/workflows/deploy-pages.yml` — build → gate → `upload-pages-artifact` of `dist/`
- `netlify.toml` — `publish = "dist"`, `command = "node scripts/build.js"`
- `vercel.json` — `outputDirectory: "dist"`, same build command

---

## 9. Invariants

Breaking any of these breaks the build contract:

1. **Never edit `dist/`** — it is wiped on the next build.
2. **Never add `<base href>`** — it would break the relative `blog/` / `notice/` / `event/`
   routes that `rebase()` produces.
3. **All internal links stay relative** — no leading `/`. This is what makes the site work at
   a sub-path with no configuration.
4. **New page body → register it in `PAGES`** — the array is the page manifest; a file in
   `src/pages/` that is not listed is never built.
5. **New asset → it must be in `LOCAL_ASSETS`** — otherwise it is silently left out of the
   cache-busting hash.
6. **New content list → JSON + renderer + `data-loader.js > VIEWS` + a `BLOCK_*` entry.**
   Adding a blog post, notice or event needs none of that: add one object with a `slug` and a
   `body` to the JSON and the page generates itself.
