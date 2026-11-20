# Next.js 16 static export on GitHub Pages: redirects, config injection, custom domain

Research date: 2026-11-19. Repo state at time of research: `next@16.2.4`, `next.config.ts` sets only `reactCompiler: true`, `.github/workflows/publish.yaml` uses `actions/configure-pages@v5` with `static_site_generator: next` and uploads `./out`, custom domain `pladypus.dev`, no `public/CNAME`.

Sources are primary only: Next.js docs bundled in `node_modules/next/dist/docs/` (authoritative for 16.2.4), Next.js compiled source in `node_modules/next/dist/`, `actions/configure-pages` source pinned to tag `v5.0.0` (= `v5`, commit [`983d773`](https://github.com/actions/configure-pages/tree/983d7736d9b0ae728b81ab479565c72886d7745b)), GitHub Docs source (`github/docs` at commit [`16d03dd`](https://github.com/github/docs/tree/16d03ddd1bd51eab7407b4ba632200618e6655c2)), Google Search Central, W3C WCAG Techniques, react.dev. Claims marked **[verified locally]** were reproduced with `next build` in this worktree (temporary changes, all reverted). Claims marked **[observed live]** come from requests to the deployed site.

## Questions

1. How should `/groomsmen/<id>` redirect to `https://groomsmen.pladypus.dev/<id>` when there's no server and `next.config` `redirects` is unsupported under `output: 'export'`? What are the SEO and UX trade-offs?
2. Does `actions/configure-pages` with `static_site_generator: next` work with a `next.config.ts` on Next 16? Should the repo set `output: 'export'` and `images.unoptimized` itself and drop `static_site_generator`?
3. With a custom Actions workflow and an apex custom domain, is a `CNAME` file needed, and what `basePath` is needed?

## Short answers

1. **Redirects.** GitHub Pages has no server-side redirect configuration, so the best option is a static HTML page per id with an instant `<meta http-equiv="refresh" content="0;url=…">` in `<head>`. Google treats an instant meta refresh as a **permanent** redirect, and it works without JavaScript. Add `<link rel="canonical">` to the target and a visible fallback link. Two details matter in Next 16. First, the Metadata API can't emit `http-equiv` (`other` renders `name=`). Second, rendering `<meta httpEquiv="refresh">` in the page JSX is hoisted into `<head>` by React 19 **[verified locally]**. Calling `redirect()`/`permanentRedirect()` at the top level of an exported page emits **no** meta tag. It produces an empty page that redirects only after JS hydration (a JS redirect, which Google ranks below meta refresh). Inside a `<Suspense>` boundary it does emit `<meta http-equiv="refresh">`: `0;url` (permanent) for `permanentRedirect`, `1;url` (delayed, so Google treats it as temporary) for `redirect` **[verified locally]**. A 404.html trick serves a 404 status, so it's worse than both.
2. **configure-pages + `next.config.ts`.** It does not inject into the `.ts` file. It looks only for `next.config.{js,cjs,mjs}`, finds none, and **creates a new `next.config.js`** containing `{images:{unoptimized:true}, basePath:"", output:"export"}`. Next resolves `next.config.js` *before* `next.config.ts`, so in CI the repo's `next.config.ts` is **silently ignored**. Today that means `reactCompiler: true` is not in effect in production **[verified locally and observed live]**. Passing the `.ts` file explicitly fails too: the parser (espree, JS only) throws on TypeScript syntax and the action only emits a warning. **Recommendation:** set `output: 'export'` (and `images.unoptimized: true` if using `next/image`) in `next.config.ts` and remove `static_site_generator: next`, which is what the official `nextjs/deploy-github-pages` example does.
3. **Custom domain.** No `CNAME` file is needed. GitHub Docs say that when publishing from a custom Actions workflow "no `CNAME` file is created, and any existing `CNAME` file is ignored and is not required", and the domain is set in repo settings or the API. The repo's Pages API already reports `cname: "pladypus.dev"`, `build_type: "workflow"`. **basePath:** none (empty). configure-pages computes it from the site URL path `/`, strips the trailing slash, and outputs `base_path` = `""` (and would inject `basePath: ""`, which equals Next's default).

---

## 1. Redirecting `/groomsmen/<id>` without a server

### 1.1 What Next.js static export does not support

- The Next.js static export guide lists **Redirects** (`next.config.js` `redirects`), Rewrites, Headers, Proxy, and Dynamic Routes without `generateStaticParams()` under "Unsupported Features". Source: `node_modules/next/dist/docs/01-app/02-guides/static-exports.md` ("How to create a static export of your Next.js application"), lines 274–292.
- `redirects` in config works at the request level. It returns 308/307 status codes, which a static file host can't produce. Source: `node_modules/next/dist/docs/01-app/03-api-reference/05-config/01-next-config-js/redirects.md` ("redirects"), lines 26–30.

### 1.2 GitHub Pages has no server-side redirect configuration

- GitHub Docs describe Pages as "a static site hosting service that takes HTML, CSS, and JavaScript files straight from a repository … and publishes a website." Source: [`content/pages/getting-started-with-github-pages/what-is-github-pages.md` L26](https://github.com/github/docs/blob/16d03ddd1bd51eab7407b4ba632200618e6655c2/content/pages/getting-started-with-github-pages/what-is-github-pages.md?plain=1#L26) (published at <https://docs.github.com/en/pages/getting-started-with-github-pages/what-is-github-pages>).
- The Pages docs have no page for configuring redirects. The old `/articles/redirects-on-github-pages` URL now redirects to "About GitHub Pages and Jekyll" ([`redirect_from` L14](https://github.com/github/docs/blob/16d03ddd1bd51eab7407b4ba632200618e6655c2/content/pages/setting-up-a-github-pages-site-with-jekyll/about-github-pages-and-jekyll.md?plain=1#L14)). That page only covers Jekyll plugins, which don't apply to an Actions-built Next export. The only Pages-level redirects GitHub documents are domain-level ones (apex ↔ `www`), described in "About custom domains and GitHub Pages" ([L33](https://github.com/github/docs/blob/16d03ddd1bd51eab7407b4ba632200618e6655c2/content/pages/configuring-a-custom-domain-for-your-github-pages-site/about-custom-domains-and-github-pages.md?plain=1#L33)). **[observed live]** `https://www.pladypus.dev/` returns `301 → https://pladypus.dev/`.
- The only customizable error behavior is a `404.html` page. Source: [`creating-a-custom-404-page-for-your-github-pages-site.md`](https://github.com/github/docs/blob/16d03ddd1bd51eab7407b4ba632200618e6655c2/content/pages/getting-started-with-github-pages/creating-a-custom-404-page-for-your-github-pages-site.md?plain=1).

**Conclusion:** per-path HTTP 301/308 is impossible on GitHub Pages. That leaves client-side mechanisms (meta refresh or JS), or moving the path to another host or a CDN in front (see 1.7).

### 1.3 Google's documented treatment of redirect types

Source: Google Search Central, "Redirects and Google Search", <https://developers.google.com/search/docs/crawling-indexing/301-redirects>:

- Server-side is preferred: "we recommend that you use a permanent server-side redirect whenever possible"; "a server side redirect has the highest chance of being interpreted correctly by Google".
- Meta refresh: "If server-side redirects aren't possible to implement on your platform, meta refresh redirects may be a viable alternative." The tag goes in `<head>`. Example: `<meta http-equiv="refresh" content="0; url=https://example.com/newlocation">`.
  - Instant (`0`) meta refresh: "Google Search interprets instant `meta refresh` redirects as permanent redirects."
  - Delayed (`>0`): "Google Search interprets delayed `meta refresh` redirects as temporary redirects."
- JavaScript `location` redirects: "Only use JavaScript redirects if you can't do server-side or `meta refresh` redirects." Google must render the page, and "rendering may fail for various reasons", so Google "might never see it if rendering of the content failed". The redirect-type table still classifies JS `location` as permanent.
- Permanent redirects are used by "the indexing pipeline … as a signal that the redirect target should be canonical"; temporary ones are not.
- "Crypto redirect" (a page with a link and explanation) is a last resort that "not all search engines may recognize".

Canonical: Google's "How to specify a canonical URL" (<https://developers.google.com/search/docs/crawling-indexing/consolidate-duplicate-urls>) calls redirects "A strong signal that the target of the redirect should become canonical" and `rel="canonical"` "A strong signal that the specified URL should become canonical". It recommends `rel="canonical"` over `noindex` for canonical selection ("We don't recommend using noindex to prevent selection of a canonical page…"). *Note:* this page does not explicitly address cross-domain canonicals. Treat a cross-domain canonical as an extra signal next to the redirect, not a substitute for it.

### 1.4 UX and accessibility treatment of meta refresh

W3C WCAG Technique H76, "Using meta refresh to create an instant client-side redirect" (<https://www.w3.org/WAI/WCAG22/Techniques/html/H76>), says the objective is "to enable redirects on the client side without confusing the user". It requires a zero time-out ("It is important that the time-out is set to zero, to avoid that content is displayed before the new page is loaded") and notes "Redirects are preferably implemented on the server side". A `0` delay is therefore the accessible choice. Delayed refreshes are the WCAG failure case.

### 1.5 Option comparison

| Option | Works without JS | HTTP status seen | Google interpretation | UX | Next 16 mechanics |
|---|---|---|---|---|---|
| **A. Static page with `<meta http-equiv="refresh" content="0;url=…">` in `<head>`** (+ canonical + visible link) | Yes | 200 | Permanent (instant meta refresh) | Near-instant; history keeps the source URL unless the browser replaces it; WCAG H76-compliant | Render `<meta httpEquiv="refresh" …/>` in page JSX; React 19 hoists it into `<head>` **[verified]** |
| **B. `permanentRedirect()` inside `<Suspense>`** in an exported page | Yes (meta emitted) | 200 | Permanent (`0;url`) | Briefly renders fallback, then refresh | Relies on the redirect being thrown in a *streaming* context; fragile (see 1.6) |
| **C. `redirect()` / `permanentRedirect()` at the top level of the page** | **No**, blank page | 200 | JS redirect: only if Google renders | Blank page until JS hydrates, then `location.replace` | Emits no meta tag **[verified]** |
| **D. Client component `useEffect(() => location.replace(url))`** | No | 200 | JS redirect | Flash of content, then redirect | Plain client JS |
| **E. No page; rely on `404.html` running JS that maps paths** | No | **404** | Not covered by Google's redirect doc. The response is a 404, so the JS redirect only counts if Google renders it, and the 404 status is a separate negative signal (not sourced here) | Flash of 404 page | Next emits `out/404.html` (static-exports.md L330–332) |
| **F. Serve the path from a host that can do HTTP 301/308** (CDN or a redirect service in front of `pladypus.dev`, or stop linking the old path) | Yes | 301/308 | Best (server-side permanent) | Best | Outside GitHub Pages; requires moving DNS for `pladypus.dev` off direct GitHub Pages hosting |

**[observed live]** `groomsmen.pladypus.dev` already resolves to GitHub Pages (`pladypus.github.io`) and serves a separate site ("War Council Collection"). The account has a `pladypus/groomsman-website` Pages repo, which presumably owns that domain (not confirmed). DNS alone can't redirect a *path* (`/groomsmen/<id>`) on `pladypus.dev` to another host. DNS works per hostname, not per path, so option F means putting something other than GitHub Pages in front of `pladypus.dev`.

### 1.6 What Next 16 actually emits for `redirect()` in an exported page

Docs: `node_modules/next/dist/docs/01-app/03-api-reference/04-functions/redirect.md` ("redirect"), line 11: "When used in a streaming context, this will insert a meta tag to emit the redirect on the client side. When used in a server action, it will serve a 303 HTTP redirect response to the caller. Otherwise, it will serve a 307 HTTP redirect response to the caller." Line 54: "`redirect` also accepts absolute URLs and can be used to redirect to external links." `permanentRedirect` is the 308 variant (line 51).

Source:
- `node_modules/next/dist/server/app-render/make-get-server-inserted-html.js` L52–61 emits `<meta id="__next-page-redirect" http-equiv="refresh" content="${isPermanent ? 0 : 1};url=${redirectUrl}">` for captured redirect errors. So permanent (308) produces an instant refresh and temporary (307) a **1-second delayed** refresh.
- `node_modules/next/dist/server/app-render/app-render.js` L4237–4320 is the prerender `catch` path for an error thrown *before* the shell is ready. It sets `res.statusCode`/`location`, which are meaningless for a static file. It renders the error shell with `serverCapturedErrors: []`, so **no meta tag is emitted**.
- Client: `node_modules/next/dist/client/components/redirect-boundary.js` calls `router.replace`/`push`. For external URLs, `router-reducer/reducers/navigate-reducer.js` L33–36 does a hard navigation, and `app-router.js` L213–222 calls `location.replace`/`location.assign`.

**[verified locally]** (temporary test routes, since deleted, built with `output: 'export'`):
- Top-level `redirect('https://groomsmen.pladypus.dev/abc')` in a page with `generateStaticParams`: the build succeeds and `out/zz-redirect-test/abc.html` has **no** `http-equiv` tag. The body is empty apart from the RSC payload, which contains the digest `NEXT_REDIRECT;replace;https://groomsmen.pladypus.dev/abc;307;`. `permanentRedirect` behaves the same, with `;308;`. The redirect therefore happens only after JS loads (option C).
- The same calls inside `<Suspense fallback="Redirecting...">` from an async child that awaits briefly: `<head>` contains `<meta id="__next-page-redirect" http-equiv="refresh" content="0;url=https://groomsmen.pladypus.dev/perm"/>` for `permanentRedirect`, and `content="1;url=…"` for `redirect` (option B). Whether the meta tag appears depends on the child actually suspending past the shell, which is why B is fragile.
- A page rendering `<meta httpEquiv="refresh" content="0;url=…"/>` directly in JSX: the tag ends up inside `<head>` of the exported HTML. React 19 docs say "React will always place the DOM element corresponding to the `<meta>` component within the document's `<head>`, regardless of where in the React tree it is rendered", and list `httpEquiv` as a supported prop (<https://react.dev/reference/react-dom/components/meta>).
- Metadata API `other: { refresh: '0;url=…' }` renders `<meta name="refresh" …>`, **not** `http-equiv`, so it does nothing. That matches the docs: `other` renders `<meta name="custom" content="meta" />` (`node_modules/next/dist/docs/01-app/03-api-reference/04-functions/generate-metadata.md`, "generateMetadata", `### other`, L1000–1014). In the same build, `alternates: { canonical }` rendered `<link rel="canonical" href="https://groomsmen.pladypus.dev/abc"/>` (doc `### alternates`, L823–844).

### 1.7 Routing facts on the live site that affect any of the options

**[observed live]**
- `https://pladypus.dev/groomsmen/best-man-of-the-eternal-jar` returns 200 (Pages serves `groomsmen/<id>.html` for the extensionless path). `…/best-man-of-the-eternal-jar/` (trailing slash) returns **404**, because the export (default `trailingSlash: false`) emits `groomsmen/<id>.html` plus a same-named directory of RSC `.txt` files with no `index.html`. The Next docs confirm `/about` → `/about.html` by default and `/about/index.html` with `trailingSlash: true` (`node_modules/next/dist/docs/01-app/03-api-reference/05-config/01-next-config-js/trailingSlash.md`, "trailingSlash", L27). GitHub Docs don't document the extensionless-to-`.html` lookup. It is only observed behavior.
- `http://pladypus.dev/...` returns 200 without redirecting to HTTPS. The Pages API reports `https_enforced: false` (`gh api repos/pladypus/pladypus.github.io/pages`).

### 1.8 Recommended redirect implementation (option A)

Keep `src/app/groomsmen/[imageId]/page.tsx` with `generateStaticParams`, and make the page render:

```tsx
const url = `https://groomsmen.pladypus.dev/${imageId}`;
// in generateMetadata: alternates: { canonical: url }
return (
  <>
    <meta httpEquiv="refresh" content={`0;url=${url}`} />
    <p>This page has moved to <a href={url}>{url}</a>.</p>
  </>
);
```

- An instant meta refresh is read by Google as permanent and passes the canonical signal (Google, 1.3). It works with JS disabled. It's WCAG H76-compliant with `0` delay (1.4).
- `rel=canonical` to the target is a second "strong signal" (1.3). Don't add `noindex` (Google advises against mixing it with canonical selection).
- Optionally add a client-side `location.replace(url)` as a belt-and-braces fallback. It's unnecessary in practice.
- Prefer this over `redirect()` (C), which gives no meta tag and a blank page without JS, and over the Suspense trick (B), which depends on streaming timing and gives a 1-second delayed refresh for `redirect()`.
- If the redirect must be a true 301/308 (best SEO), GitHub Pages can't do it (1.2). It needs a CDN or proxy in front of `pladypus.dev`.

---

## 2. `actions/configure-pages` `static_site_generator: next` vs `next.config.ts`

### 2.1 What the action does (v5 = v5.0.0 = commit `983d773`)

- `action.yml` inputs: `static_site_generator` ("nuxt", "next", "gatsby", or "sveltekit") and optional `generator_config_file`. Output `base_path` ("Examples: \"/my-repo\" or \"\""). Runtime `node20`. Source: [`action.yml`](https://github.com/actions/configure-pages/blob/983d7736d9b0ae728b81ab479565c72886d7745b/action.yml).
- `src/index.js` gets the Pages site via the API, builds `siteUrl = new URL(pageObject.html_url)`, and calls `setPagesConfig` only if `staticSiteGenerator` is set. It then always calls `outputPagesBaseUrl`. Source: [`src/index.js` L12–20](https://github.com/actions/configure-pages/blob/983d7736d9b0ae728b81ab479565c72886d7745b/src/index.js#L12-L20).
- **File names / extensions:** `SUPPORTED_FILE_EXTENSIONS = ['.js', '.cjs', '.mjs']`. `detectOrDefaultConfigFile('next.config')` returns the first existing `./next.config{.js,.cjs,.mjs}`, **otherwise `./next.config.js`**. `.ts` is never probed. Source: [`src/set-pages-config.js` L7–18, L44](https://github.com/actions/configure-pages/blob/983d7736d9b0ae728b81ab479565c72886d7745b/src/set-pages-config.js#L7-L18).
- **What it injects for `next`:** `output: 'export'`, `basePath: <site path without trailing slash>`, `'images.unoptimized': true`. Source: [`src/set-pages-config.js` L39–59](https://github.com/actions/configure-pages/blob/983d7736d9b0ae728b81ab479565c72886d7745b/src/set-pages-config.js#L39-L59).
- **Missing file behavior:** if the config file doesn't exist, `ConfigParser` writes the blank template there and logs "Using default blank configuration". The template is `const nextConfig = {}` / `module.exports = nextConfig`. Source: [`src/config-parser.js` L89–96](https://github.com/actions/configure-pages/blob/983d7736d9b0ae728b81ab479565c72886d7745b/src/config-parser.js#L89-L96), [`src/blank-configurations/next.js`](https://github.com/actions/configure-pages/blob/983d7736d9b0ae728b81ab479565c72886d7745b/src/blank-configurations/next.js).
- **Parser:** `espree.parse(config, { ecmaVersion: 'latest', sourceType: 'module', range: true })` (espree `^9.6.1` per `package.json`). It looks for an object literal in `export default {…}`, `export default ident`, or `module.exports = …`, then splices text by AST ranges. Wrapping calls are only allowed when `allowWrappingCall` is set, and the `next` settings don't set it. espree parses JavaScript only. There's no Babel or TypeScript support. Source: [`src/config-parser.js` L138–281, L325–341](https://github.com/actions/configure-pages/blob/983d7736d9b0ae728b81ab479565c72886d7745b/src/config-parser.js#L325-L341).
- **Unsupported extension / parse failure:** passing a non-JS `generator_config_file` produces a `core.warning(...)` ("We will still attempt…"). Any exception during parsing is caught and becomes a warning ("We were unable to determine how to inject the site metadata into your config…"). It never fails the step. Source: [`src/set-pages-config.js` L91–110](https://github.com/actions/configure-pages/blob/983d7736d9b0ae728b81ab479565c72886d7745b/src/set-pages-config.js#L91-L110).
- **v6.0.0** (2026-03-25, commit `45bfe01`) changes the runtime to `node24`. `src/set-pages-config.js`, `src/config-parser.js` and `src/index.js` are byte-identical to v5 (diffed locally). Release notes: <https://github.com/actions/configure-pages/releases/tag/v6.0.0>. Upgrading doesn't fix the TS issue.
- **Open issues:** [#177 "Add support for Typescript configurations"](https://github.com/actions/configure-pages/issues/177) (open, 2025-05-25, explicitly mentions `next.config.ts` and the espree limitation). [#74 "The action ignores .ts config and creates default nuxt.config.js"](https://github.com/actions/configure-pages/issues/74) (open). [#28](https://github.com/actions/configure-pages/issues/28) (closed 2022, same problem for `gatsby-config.ts`).

### 2.2 How Next 16 picks the config file

- `CONFIG_FILES = ['next.config.js', 'next.config.mjs', 'next.config.ts', (+ 'next.config.mts' if Node has native TS)]`. Source: `node_modules/next/dist/shared/lib/constants.js` L359–367.
- `loadConfig` calls `findUp(CONFIG_FILES, { cwd: dir })` and uses the first match, so **`next.config.js` wins over `next.config.ts`** when both exist. Source: `node_modules/next/dist/server/config.js` L1202–1208.
- `next.config.ts` is officially supported. Source: `node_modules/next/dist/docs/01-app/03-api-reference/05-config/02-typescript.md` ("TypeScript"), L98–110; `…/01-next-config-js/index.md` L102–104.

### 2.3 Reproduction

**[verified locally]**
1. Running the v5 `setPagesConfig` source (copied to a scratch dir with `espree@9.6.1` and `@actions/core`) in a directory containing only this repo's `next.config.ts`, with `siteUrl = http://pladypus.dev/`, created `next.config.js`:
   ```js
   // Default Pages configuration for Next
   const nextConfig = {images: {unoptimized: true},basePath: "",output: "export"}
   module.exports = nextConfig
   ```
   `next.config.ts` was left untouched.
2. Running it with `generator_config_file: ./next.config.ts` printed `::warning::Unsupported extension in configuration file: ./next.config.ts…` and then `::warning title=SyntaxError…::We were unable to determine how to inject… Error: Unexpected token {` (espree choking on `import type { NextConfig }`). No file was changed.
3. `next build` in the worktree with that generated `next.config.js` next to the original `next.config.ts` succeeded with a static export to `out/`. But the client chunk containing `scratch-off.tsx` had **zero** React Compiler cache calls (`(0,x.c)(n)`). With `next.config.ts` alone plus `output: 'export'`, the same chunk contained `(0,r.c)(5)`. So the `.ts` config, including `reactCompiler: true`, is ignored when the injected `.js` exists.

**[observed live]** The deployed `https://pladypus.dev/groomsmen/best-man-of-the-eternal-jar` loads `/_next/static/chunks/0hs5dvyr6f55s.js`, the same content-hashed chunk name produced by the local "injected `next.config.js`" build. It has zero React Compiler cache calls. **Production currently builds without the React Compiler.**

### 2.4 What the official Next.js example does

The Next docs' deploying guide links GitHub Pages to `https://github.com/nextjs/deploy-github-pages` (`node_modules/next/dist/docs/01-app/01-getting-started/17-deploying.md` L74). That example (commit [`8cef01d`](https://github.com/nextjs/deploy-github-pages/tree/8cef01dd18d210683452e7ce434ce776235e0779)):
- `next.config.ts`: `{ output: 'export', basePath: process.env.PAGES_BASE_PATH }`.
- `.github/workflows/deploy.yml`: `uses: actions/configure-pages@v5` with `id: setup_pages` and **no `static_site_generator`**. The build step passes `PAGES_BASE_PATH: ${{ steps.setup_pages.outputs.base_path }}`.

The GitHub starter workflow (`actions/starter-workflows/pages/nextjs.yml`, which this repo's workflow matches) still sets `static_site_generator: next`, with the comment "You may remove this line if you want to manage the configuration yourself."

### 2.5 Is `images.unoptimized` needed?

- The static export guide lists "Image Optimization with the default `loader`" as unsupported (`static-exports.md` L289). It offers a custom loader as the alternative (L168–229).
- `images: { unoptimized: true }` in `next.config` applies `unoptimized` to all images ("Since Next.js 12.3.0, this prop can be assigned to all images…"). Source: `node_modules/next/dist/docs/01-app/03-api-reference/02-components/image.md` ("Image Component"), `#### unoptimized`, L391–412.
- `src/components/yugioh-card.tsx` imports `next/image`, so the repo needs it. **[verified locally]** The build with `output: "export"` and `images: { unoptimized: true }` in `next.config.ts` succeeded. (Not tested here: the build *without* `unoptimized`, where the docs say the default loader is unsupported.)

### 2.6 Conclusion for Q2

Yes. Set `output: 'export'` and `images: { unoptimized: true }` explicitly in `next.config.ts`, keep `reactCompiler: true`, and remove `static_site_generator: next` from `publish.yaml`. Keep `actions/configure-pages` itself, which still enables Pages and provides outputs. With the line removed, the action no longer writes a `next.config.js`, so Next reads the `.ts` file. `basePath` can be left unset (see Q3), or wired from `steps.<id>.outputs.base_path` as in the official example.

---

## 3. Custom domain and basePath

### 3.1 CNAME file

- "If you are publishing from a custom GitHub Actions workflow, no `CNAME` file is created, and any existing `CNAME` file is ignored and is not required." Source: GitHub Docs "Managing a custom domain for your GitHub Pages site", [L47 (apex) and L117 (subdomain)](https://github.com/github/docs/blob/16d03ddd1bd51eab7407b4ba632200618e6655c2/content/pages/configuring-a-custom-domain-for-your-github-pages-site/managing-a-custom-domain-for-your-github-pages-site.md?plain=1#L47) (published at <https://docs.github.com/en/pages/configuring-a-custom-domain-for-your-github-pages-site/managing-a-custom-domain-for-your-github-pages-site>).
- "A `CNAME` file in your repository file does not automatically add or remove a custom domain. Instead, you must configure the custom domain through your repository settings or through the API." Source: "Configuring a publishing source for your GitHub Pages site", [L83](https://github.com/github/docs/blob/16d03ddd1bd51eab7407b4ba632200618e6655c2/content/pages/getting-started-with-github-pages/configuring-a-publishing-source-for-your-github-pages-site.md?plain=1#L83).
- Current repo state: `gh api repos/pladypus/pladypus.github.io/pages` returns `"cname":"pladypus.dev"`, `"build_type":"workflow"`, `"protected_domain_state":"verified"`, `"html_url":"http://pladypus.dev/"`. The domain is configured in settings, so no `public/CNAME` is needed.

### 3.2 basePath

- `basePath` is only for deploying "under a sub-path of a domain"; the default is `''`. Source: `node_modules/next/dist/docs/01-app/03-api-reference/05-config/01-next-config-js/basePath.md` ("basePath"), L8–18. The site is served at the root of `pladypus.dev`, so no basePath is needed.
- configure-pages derives it from the Pages `html_url`. `siteUrl.pathname` for `http://pladypus.dev/` is `/`. `removeTrailingSlash('/')` gives `''`. That value is output as `base_path` ([`src/output-pages-base-url.js` L4–12](https://github.com/actions/configure-pages/blob/983d7736d9b0ae728b81ab479565c72886d7745b/src/output-pages-base-url.js#L4-L12)) and injected as `basePath: ""` when `static_site_generator: next` is set ([`set-pages-config.js` L39–53](https://github.com/actions/configure-pages/blob/983d7736d9b0ae728b81ab479565c72886d7745b/src/set-pages-config.js#L39-L53); [`remove-trailing-slash.js`](https://github.com/actions/configure-pages/blob/983d7736d9b0ae728b81ab479565c72886d7745b/src/remove-trailing-slash.js)). **[verified locally]** The simulation produced `basePath: ""`.
- A project site *without* a custom domain would get `/repo-name`. This is a user site (`pladypus.github.io`) with an apex custom domain, so the base path is empty either way.

---

## Recommendation

1. `next.config.ts`: add `output: 'export'` and `images: { unoptimized: true }`, keep `reactCompiler: true`. Optionally add `trailingSlash: true` so `/groomsmen/<id>/` also resolves (it then emits `<id>/index.html`). Don't add `basePath`.
2. `.github/workflows/publish.yaml`: remove `with: static_site_generator: next` from the `actions/configure-pages` step. This stops the generated `next.config.js` from shadowing `next.config.ts`, which also restores the React Compiler in production. Optionally bump to `@v6` (Node 24 runtime, same logic).
3. Don't add `public/CNAME`. The domain is set in repo settings and a CNAME file is ignored for Actions deployments.
4. Groomsmen redirect: change `src/app/groomsmen/[imageId]/page.tsx` to render `<meta httpEquiv="refresh" content="0;url=https://groomsmen.pladypus.dev/<id>">`, a visible link, and `alternates.canonical` to the target. Keep `generateStaticParams`. Don't rely on `redirect()` from `next/navigation` for this.
5. Consider enabling "Enforce HTTPS" in Pages settings (`https_enforced` is currently `false`).

## Open questions

- Is a true HTTP 301/308 worth fronting `pladypus.dev` with a CDN or proxy? GitHub Pages alone can't do it.
- Are the old `/groomsmen/<id>` URLs indexed or linked externally? If not, SEO impact is moot and the meta-refresh page only matters for UX (shared links, QR codes).
- GitHub doesn't document the extensionless `/x` → `x.html` lookup. It was observed live, not documented.
- Option B (Suspense + `permanentRedirect`) was verified only with an artificially suspending child (50 ms await). It wasn't tested whether a non-suspending async child still produces the meta tag.
- Not verified in a real browser: that the option C exported page actually navigates via `location.replace` after hydration. That claim is based on the source reading in 1.6.
- Ownership of `groomsmen.pladypus.dev` (presumably `pladypus/groomsman-website`) was not confirmed via the Pages API.
