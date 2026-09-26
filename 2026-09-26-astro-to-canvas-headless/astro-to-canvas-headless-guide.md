# From a fresh Astro app to Canvas Headless compatibility

This guide explains every step that separates a freshly scaffolded Astro app
from the Astro template in the
[Canvas headless templates repository](https://github.com/drupal-canvas/headless-templates)
(its `astro/` directory), so that an existing Astro project can be made Canvas
Headless compatible by hand. It was written on 2026-09-26 by scaffolding a clean
Astro app with the official installer, applying the steps below to a copy of it,
and comparing the result with this template and with the SDK sources it
consumes.

Every step is marked with one of three labels:

- **Tested**: executed in this exercise, with the command and result recorded in
  the [verification log](#verification-log).
- **Inferred**: read from the SDK, template, or module sources (paths are given)
  but not executed here.
- **Blocked**: could not be executed here. The first pass ran against a mock
  backend only; a second pass on the same day installed a local Drupal site with
  the `canvas_headless` module and verified the editor integration end to end
  (see [Real Drupal validation](#real-drupal-validation)). Items still marked
  Blocked were not exercised even then.

Paths starting with `packages/` or `modules/` refer to the
[Drupal Canvas repository](https://git.drupalcode.org/project/canvas). Paths
starting with `astro/` refer to that template directory in the headless
templates repository; "the template" below always means it. Paths under
`/home/hazelnut/workspace/astro-comparison/` are the scratch work area of the
exercise and are not part of any repository.

## Contents

1. [Versions at a glance](#versions-at-a-glance)
2. [Step 0: scaffold a fresh Astro app](#step-0-scaffold-a-fresh-astro-app)
3. [What the template adds: file-by-file comparison](#what-the-template-adds-file-by-file-comparison)
4. [Minimum required steps](#minimum-required-steps)
5. [Optional steps](#optional-steps)
6. [Drupal-side configuration](#drupal-side-configuration)
7. [How the integration works at runtime](#how-the-integration-works-at-runtime)
8. [Build and deployment](#build-and-deployment)
9. [Verification log](#verification-log)
10. [Known differences and pitfalls](#known-differences-and-pitfalls)
11. [Daytona-only workarounds (not applied)](#daytona-only-workarounds-not-applied)

## Versions at a glance

Resolved on 2026-09-26 with Node.js 24.21.0 and npm 11.18.0. "Fresh Astro" is
what `npm create astro@latest` produced; "Template lockfile" is what `npm ci`
installs from `astro/package-lock.json`; "Template range" is the range in
`astro/package.json`.

| Package                             | Fresh Astro (latest)             | Template lockfile | Template range |
| ----------------------------------- | -------------------------------- | ----------------- | -------------- |
| `astro`                             | 7.3.5                            | 7.1.6             | `^7.1.6`       |
| `vite` (transitive)                 | 8.3.1                            | 8.1.5             | n/a            |
| `typescript`                        | 6.0.3 (added by hand)            | 5.9.3             | `^5.9.3`       |
| `@astrojs/check`                    | 0.9.10 (added by hand)           | 0.9.10            | `^0.9.6`       |
| `@astrojs/node`                     | 11.1.6 (added by hand)           | 11.0.3            | `^11.0.2`      |
| `@drupal-canvas/headless-astro`     | 0.7.0 (added by hand)            | 0.7.0             | `^0.7.0`       |
| `@drupal-canvas/headless`           | 0.11.0 (added by hand)           | 0.11.0            | `^0.11.0`      |
| `drupal-canvas` (transitive)        | 0.7.1                            | 0.7.1             | n/a            |
| `unhead`                            | 3.4.1 (added by hand)            | 3.3.1             | `^3.3.1`       |
| `tailwindcss` / `@tailwindcss/vite` | 4.3.3 (via `astro add tailwind`) | 4.3.3             | `^4.1.18`      |
| `create-astro` (installer)          | 5.2.4 (npm `latest` at run time) | n/a               | n/a            |

Two version facts matter for the steps below:

- The SDK adapter declares `astro >=5.17.0` and `vite >=6.0.0` as peer
  dependencies (`packages/headless-astro/package.json`), so the current Astro
  major works without changes to the adapter.
- `astro check` reports a type error on the template's `Astro.redirect()` call
  in a fresh install but not with the template's lockfile. The Astro version is
  not the cause: both 7.1.6 and 7.3.5 declare
  `redirect(path, status?: ValidRedirectStatus)`. The difference is the
  `@astrojs/language-server` that `@astrojs/check` 0.9.10 resolves: 2.16.13 in
  the template lockfile stays silent, 2.17.1 in a fresh install reports
  `ts(2345)`. Swapping only that package flips the result in both directions
  (tested). See [step 5](#5-replace-the-index-page-with-a-catch-all-route) for
  the narrowing that satisfies both.

The SDK packages require Node.js `>=22.19.0 <23 || >=24.5.0`
(`packages/headless-astro/package.json`), which is stricter than fresh Astro's
`>=22.12.0`. The template copies the SDK range into its own `engines`.

## Step 0: scaffold a fresh Astro app

**Tested.** The official installation page
(<https://docs.astro.build/en/install-and-setup/>) documents the CLI wizard:

```bash
npm create astro@latest
```

The wizard asks for a directory, a template, whether to install dependencies,
and whether to initialize git. The flags the installer documents (`--template`,
`--install`/`--no-install`, `--git`/`--no-git`, `--yes`, `--no`, `--add`,
`--skip-houston`, `--no-ai`, `--dry-run`, `--ref`) let you run it unattended.
The exact command used for the baseline in this exercise, run from
`/home/hazelnut/workspace/astro-comparison/`:

```bash
npm create astro@latest baseline -- --yes --no-git --install
```

`--yes` accepted every default: the `basics` template, dependency install with
npm, and AI agent files. The scaffold took about 15 seconds. The result is
preserved untouched at `/home/hazelnut/workspace/astro-comparison/baseline/` and
consists of:

```text
baseline/
├── .gitignore
├── .vscode/extensions.json
├── .vscode/launch.json
├── AGENTS.md              # generated by the --yes default (AI agent files)
├── CLAUDE.md -> AGENTS.md
├── README.md
├── astro.config.mjs       # defineConfig({})
├── package.json           # only dependency: astro ^7.3.5
├── package-lock.json
├── public/favicon.ico
├── public/favicon.svg
├── src/assets/astro.svg
├── src/assets/background.svg
├── src/components/Welcome.astro
├── src/layouts/Layout.astro
├── src/pages/index.astro
└── tsconfig.json          # extends astro/tsconfigs/strict
```

Facts about this baseline that the migration has to deal with:

- `astro.config.mjs` is empty, which means `output: 'static'`: every page is
  prerendered at build time and there is no adapter. Canvas needs per-request
  rendering.
- `src/pages/index.astro` serves `/`. Canvas serves `/` through Drupal, so this
  file collides with the catch-all route and has to go.
- `src/components/Welcome.astro` lives in the directory Canvas discovery scans.
  It has no `component.yml`, so discovery ignores it, but it is dead code once
  the index page is gone.
- The baseline builds as a static site (`npm run build` produced
  `dist/index.html`; tested in a scratch copy so the baseline stayed pristine).
- `package.json` carries `"allowScripts": { "esbuild": true }`, npm 11's
  build-script approval for esbuild. The template solves the same problem for
  pnpm with `pnpm-workspace.yaml` (`allowBuilds: esbuild: true`).

## What the template adds: file-by-file comparison

The template directory `astro/` compared with the fresh baseline. "Required"
means the Canvas integration does not work without it.

| Path                                                                  | Baseline                     | Template                                                                  | Required?                           | Purpose                                                                           |
| --------------------------------------------------------------------- | ---------------------------- | ------------------------------------------------------------------------- | ----------------------------------- | --------------------------------------------------------------------------------- |
| `astro.config.mjs`                                                    | empty config                 | server output, Node adapter, `canvas()` integration, Tailwind Vite plugin | Required (Tailwind optional)        | [Step 2](#2-configure-astro-for-per-request-rendering-and-the-canvas-integration) |
| `canvas.config.json`                                                  | absent                       | component dir, global CSS path, sync flags                                | Required                            | [Step 3](#3-add-canvasconfigjson)                                                 |
| `.env.example`                                                        | absent                       | `CANVAS_SITE_URL`, CLI credentials                                        | Required (`CANVAS_SITE_URL` only)   | [Step 4](#4-set-canvas_site_url)                                                  |
| `src/pages/index.astro`                                               | Welcome page                 | absent                                                                    | Must be removed                     | [Step 5](#5-replace-the-index-page-with-a-catch-all-route)                        |
| `src/pages/[...slug].astro`                                           | absent                       | catch-all Drupal route                                                    | Required                            | [Step 5](#5-replace-the-index-page-with-a-catch-all-route)                        |
| `src/layouts/Layout.astro`                                            | static head, `<slot />`      | Drupal head tags, `DraftBanner`, global CSS import                        | Required (shape is yours)           | [Step 6](#6-render-drupals-document-head-in-the-layout)                           |
| `src/components/DraftBanner.astro`                                    | absent                       | banner slotted into `DraftSession.astro`                                  | Required for draft preview          | [Step 7](#7-add-the-draft-session-banner)                                         |
| `src/components/*/component.yml` + `index.astro`                      | `Welcome.astro` only         | 18 Canvas components                                                      | At least one is needed to be useful | [Step 8](#8-add-canvas-components)                                                |
| `src/components/*/mocks.json`                                         | absent                       | Workbench mocks                                                           | Not used by Astro                   | [Optional](#optional-steps)                                                       |
| `src/styles/global.css`                                               | absent                       | Tailwind entry with `@theme` tokens                                       | File must exist; Tailwind optional  | [Step 3](#3-add-canvasconfigjson), [Optional](#tailwind-css)                      |
| `src/middleware.ts`                                                   | absent                       | `trustSystemCertificates()`                                               | Optional (local HTTPS)              | [Optional](#trusting-local-https-certificates)                                    |
| `src/lib/content.ts`                                                  | absent                       | JSON:API listing helpers (reference only)                                 | Optional                            | [Runtime](#jsonapi-access-from-server-code)                                       |
| `.gitignore`                                                          | Astro defaults               | adds `/.canvas/`, `.env.*`                                                | Recommended                         | [Step 9](#9-ignore-build-artifacts)                                               |
| `package.json` scripts                                                | dev/build/preview/astro      | adds `lint`, `type-check`, `check`; `engines` from SDK                    | Optional                            | [Optional](#linting-type-checking-and-engines)                                    |
| `eslint.config.js`                                                    | absent                       | `@drupal-canvas/eslint-config` + `eslint-plugin-astro`                    | Optional                            | [Optional](#linting-type-checking-and-engines)                                    |
| `pnpm-workspace.yaml`                                                 | absent                       | esbuild build-script approval for pnpm                                    | Optional                            | [Optional](#package-manager-notes)                                                |
| `.agents/skills/*`, `skills-lock.json`                                | absent (`AGENTS.md` instead) | Canvas agent skills                                                       | Optional                            | [Optional](#agent-skills)                                                         |
| `src/assets/*`, `Welcome.astro`, `.vscode/`, `AGENTS.md`, `README.md` | present                      | absent                                                                    | Remove or keep                      | Not Canvas related                                                                |
| `public/favicon.svg`                                                  | present                      | absent                                                                    | Keep or remove                      | Not Canvas related                                                                |

## Minimum required steps

The steps were applied, in this order, to a copy of the baseline at
`/home/hazelnut/workspace/astro-comparison/migrated/`. That copy is the
reference for every file shown below.

### 1. Install the SDK and the Node adapter

**Tested.**

```bash
npm install @astrojs/node @drupal-canvas/headless-astro @drupal-canvas/headless unhead
npm install -D @astrojs/check typescript
```

Why each package:

- `@drupal-canvas/headless-astro`: the Astro adapter. It ships TypeScript
  source, not compiled output, and its `canvas()` integration bundles itself and
  the core into the server build (`vite.ssr.noExternal`) so no consumer tooling
  change is needed (`packages/headless-astro/src/integration.ts`).
- `@drupal-canvas/headless`: the framework-agnostic core. It is a dependency of
  the adapter already, but the template lists it directly because app code
  imports from it (`@drupal-canvas/headless/node` in the middleware, and any
  helper you use from the isomorphic entry). Listing it also pins the version
  your app was written against.
- `@astrojs/node`: draft preview needs a request-time server. The adapter README
  names `@astrojs/node` "or equivalent"; the template uses standalone mode.
- `unhead`: not required by the SDK. `fetchPage()` returns `page.head` in
  Unhead's input shape (`packages/headless/src/page.ts`, `PageHead`), and the
  template renders it with `createHead()` from `unhead/server`. You may render
  the same object with any head manager or by hand.
- `@astrojs/check` and `typescript`: only for `astro check`. Fresh Astro does
  not install them.

Resolved versions after these installs are in the table above. Note that plain
`npm install -D typescript` resolved TypeScript 6.0.3, one major above the
template's 5.9.3; `astro check` passed with it after the redirect fix in step 5,
and the TypeScript major made no difference to that check (tested with 5.9.3 as
well).

### 2. Configure Astro for per-request rendering and the Canvas integration

**Tested.** Replace `astro.config.mjs`:

```js
// @ts-check
import node from '@astrojs/node';
import canvas from '@drupal-canvas/headless-astro/integration';
import { defineConfig } from 'astro/config';

export default defineConfig({
  output: 'server',
  adapter: node({ mode: 'standalone' }),
  integrations: [canvas()],
});
```

Rationale, from `packages/headless-astro/src/integration.ts` and the
[on-demand rendering guide](https://docs.astro.build/en/guides/on-demand-rendering/):

- `output: 'server'` renders every page on demand. The default `'static'`
  prerenders pages at build time, which can never show draft content and cannot
  read the draft session cookie. The adapter README states that pages showing
  draft content "must not be prerendered". You could keep `'static'` and set
  `export const prerender = false` on the catch-all page instead; the routes the
  integration injects already set `prerender = false` themselves, so they work
  from a static project too.
- `adapter: node({ mode: 'standalone' })` produces `dist/server/entry.mjs`, a
  self-starting Node server. Middleware mode works as well if you embed the
  handler in your own server.
- `canvas()` is the whole integration. On `astro:config:setup` it:
  - bridges `CANVAS_SITE_URL`, `CANVAS_EDITOR_ORIGINS`, `CANVAS_JSONAPI_PREFIX`,
    `CANVAS_JSONAPI_URL`, `CANVAS_JSONAPI_SITE_URL` and
    `CANVAS_JSONAPI_PROXY_PATH` from Astro's `.env` files into `process.env`,
    because the core reads `process.env`, not `import.meta.env`;
  - in `astro dev`, fails fast if `CANVAS_SITE_URL` is unset;
  - in `astro dev`, sets `security.allowedDomains` to allow any host and turns
    Vite's own CORS middleware off, so the metadata endpoint can answer the
    Canvas editor's cross-origin fetch with its own CORS contract;
  - registers the Vite plugin that generates the component registry
    (`virtual:@drupal-canvas/headless/components`) and the plugin that inlines
    the build-time manifest (`virtual:@drupal-canvas/headless-astro/manifest`);
  - adds `vite.ssr.noExternal` for the two SDK packages;
  - registers a `pre` middleware that merges a `frame-ancestors` directive into
    every response's `Content-Security-Policy`;
  - injects these routes (all `prerender = false`):
    - `GET /api/canvas/component-preview` (isolated one-component preview,
      always injected),
    - `GET /api/draft` (draft activation),
    - `POST /api/draft/renew` (in-place renewal),
    - `POST /api/disable-draft` (exit),
    - `GET|OPTIONS /api/canvas/components` (component metadata; path
      configurable with `componentsRoutePath`),
    - `ALL /api/canvas/jsonapi/[...path]` (same-origin JSON:API proxy; base path
      from `CANVAS_JSONAPI_PROXY_PATH`).

  Pass `canvas({ injectRoutes: false })` to mount the `routes/*` subpath exports
  at paths of your own. On `astro:build:start` the integration runs component
  discovery and writes `.canvas/components.manifest.json`; a malformed
  `component.yml` fails the build.

### 3. Add `canvas.config.json`

**Tested.** Create `canvas.config.json` in the project root:

```json
{
  "componentDir": "src/components",
  "globalCssPath": "src/styles/global.css",
  "sync": {
    "pages": false,
    "contentTemplates": false
  }
}
```

And create the file it points at, `src/styles/global.css` (plain CSS is enough;
see [Tailwind](#tailwind-css) for the template's version):

```css
html {
  font-family: Inter, Roboto, 'Helvetica Neue', Arial, sans-serif;
  color: #020617;
}
body {
  margin: 0;
}
```

Rationale (`packages/discovery/src/config.ts`,
`packages/headless/src/component-registry.ts`,
`packages/headless-astro/src/components/ComponentPreviewPage.astro`):

- Discovery resolves `canvas.config.json` from the Astro project root. If the
  file is missing, the defaults apply: `componentDir: 'src/components'`,
  `globalCssPath: 'src/global.css'`, `pagesDir: 'pages'`, and `sync.pages`,
  `sync.contentTemplates`, `sync.pageTemplates` all `true`. The template states
  the values explicitly.
- `globalCssPath` must exist. The injected component preview page imports
  `virtual:@drupal-canvas/headless/global.css`, which the registry plugin
  resolves to this path. The template keeps it at `src/styles/global.css`; fresh
  Astro has no global stylesheet at all.
- `sync.pages` and `sync.contentTemplates` are Canvas CLI concerns (page and
  content template specs pushed by `canvas push`). The template disables them
  because it ships no `pages/` or `content-templates/` directories. The template
  also carries `"regions": false`, a key discovery only parses as a legacy hint
  (`legacy.syncRegions`); it is not needed in a new project.

### 4. Set `CANVAS_SITE_URL`

**Tested.** Create `.env` (git-ignored) and `.env.example` (committed):

```bash
# .env
CANVAS_SITE_URL=https://your-drupal-site.example.com
```

```bash
# .env.example
# Base URL of the Drupal site, without a trailing slash.
CANVAS_SITE_URL=https://example.com
```

Rationale (`packages/headless/src/server/config.ts`):

- `CANVAS_SITE_URL` is the only required setting. It is the base URL the app's
  server calls for `/canvas/content-api`, `/canvas/api/v0/site-data`,
  `/oauth/token` and JSON:API. There is no client secret: the OAuth client id is
  the constant `canvas_headless`, a public client provisioned by the Drupal
  module, and the credential is a single-use signed assertion Drupal mints per
  preview (`packages/headless/src/constants.ts`).
- The template's `.env.example` also lists `CANVAS_CLIENT_ID` and
  `CANVAS_CLIENT_SECRET`. Those belong to the optional Canvas CLI
  (`@drupal-canvas/cli`), not to the SDK.
- Optional variables: `CANVAS_EDITOR_ORIGINS` (replaces the default
  `frame-ancestors` sources), `CANVAS_JSONAPI_PREFIX`, `CANVAS_JSONAPI_URL`,
  `CANVAS_JSONAPI_SITE_URL`, `CANVAS_JSONAPI_PROXY_PATH` (default
  `/api/canvas/jsonapi`).
- `CANVAS_JSONAPI_PROXY_PATH` is read twice: the integration mounts the proxy
  route at that path when the config loads (`astro dev` or `astro build`;
  `packages/headless-astro/src/integration.ts`), and the core reads it again
  from `process.env` on every request to decide which request paths it accepts.
  The build-time and runtime values must match. Tested: a build with `/proxy-x`
  run without the variable answered `404 endpoint_not_allowed` on `/proxy-x/...`
  and a plain 404 on the default path; the same build run with the variable
  answered 200. Rebuild after changing it.
- `.env` is read by `astro dev` and `astro build` only. The built server reads
  `process.env` at request time and does not load `.env`. Tested: the built
  server started without the variable answered `500 configuration_error` on the
  metadata endpoint. Supply the variable to the production process.

### 5. Replace the index page with a catch-all route

**Tested.** Remove the starter files:

```bash
rm src/pages/index.astro src/components/Welcome.astro src/assets/astro.svg src/assets/background.svg
rmdir src/assets
```

`index.astro` must go: Astro routes `/` to it in preference to `[...slug]`, so
Drupal's front page would never render. The other files are unused once the
index page is gone (discovery ignores `Welcome.astro` because it has no
`component.yml`, so keeping it is harmless but pointless).

Create `src/pages/[...slug].astro`:

```astro
---
import CanvasComponentTree from '@drupal-canvas/headless-astro/CanvasComponentTree.astro';
import { fetchPage, isPageRedirect } from '@drupal-canvas/headless-astro';
import { createHead } from 'unhead/server';
import Layout from '../layouts/Layout.astro';

import type { PageHead } from '@drupal-canvas/headless-astro';

type AstroHead = Omit<PageHead, 'title'> & { title?: string };

const { slug } = Astro.params;
const path = `/${(slug ?? '').split('/').map(encodeURIComponent).join('/')}`;
const result = await fetchPage(Astro, path);

if (result && isPageRedirect(result)) {
  // Astro.redirect() accepts only the redirect status codes; Drupal's
  // configured statusCode arrives as a plain number, so narrow it.
  const status = result.redirect.statusCode;
  return Astro.redirect(
    result.redirect.url,
    status === 301 || status === 302 || status === 303 || status === 307 || status === 308 ? status : 302,
  );
}

const page = result;
if (!page) {
  Astro.response.status = 404;
}

const head = createHead<AstroHead>();
head.push({ link: [{ rel: 'icon', href: '/favicon.ico' }] });
head.push(page?.head ?? { title: 'Not found' });
const { headTags } = head.render();
---

<Layout headTags={headTags}>
  {
    page ? (
      <CanvasComponentTree tree={page.content} />
    ) : (
      <main>
        <h1>Not found</h1>
        <p>Drupal answered nothing for <code>{path}</code>.</p>
      </main>
    )
  }
</Layout>
```

This is the template's `astro/src/pages/[...slug].astro` with two changes: the
redirect status narrowing and plain markup instead of Tailwind classes in the
not-found branch. Rationale:

- `[...slug]` is Astro's rest parameter: it matches `/` (slug `undefined`) and
  every deeper path. The route hands the path to Drupal's own routing through
  `fetchPage(Astro, path)`, so aliases, `/node/1`, revision and preview paths,
  and language prefixes all resolve on the Drupal side
  (`packages/headless/src/server/content-api.ts`, which calls
  `GET {CANVAS_SITE_URL}/canvas/content-api?requestUri=...`).
- Every SDK accessor takes the `Astro` global (or the `APIContext` in
  endpoints), because Astro exposes cookies per request; the draft session lives
  in those cookies (`packages/headless-astro/src/server.ts`).
- `fetchPage()` returns `null` for 403 and 404 answers, a `PageRedirect` when
  Drupal resolved a configured redirect, and otherwise a `Page` with `content`,
  `head`, `route`, and `context`. `content` is `null` when Canvas does not
  manage the route; the tree renderer accepts `null`.
- The redirect narrowing satisfies `Astro.redirect()`'s signature,
  `(path, status?: ValidRedirectStatus)`, which is the same in Astro 7.1.6 and
  7.3.5. The template's unnarrowed call is a genuine type error that the
  `@astrojs/language-server` 2.16.13 in the template lockfile does not report
  and the 2.17.1 a fresh install resolves does (tested by swapping only that
  package). The narrowing is also the safer runtime behaviour, since Drupal's
  `statusCode` is configurable.
- A failed network call to Drupal is not caught by the SDK (there is no
  `try`/`catch` around the fetch in `content-api.ts`), so an unreachable
  `CANVAS_SITE_URL` surfaces as an Astro 500 error page. Wrap the call if you
  want a softer failure.

### 6. Render Drupal's document head in the layout

**Tested.** Replace `src/layouts/Layout.astro`:

```astro
---
import DraftBanner from '../components/DraftBanner.astro';
import '../styles/global.css';

interface Props {
  headTags?: string;
}
const { headTags } = Astro.props;
---

<!doctype html>
<html lang="en">
  <head>
    {
      headTags ? (
        <Fragment set:html={headTags} />
      ) : (
        <>
          <meta charset="utf-8" />
          <meta name="viewport" content="width=device-width" />
          <link rel="icon" href="/favicon.ico" />
          <title>Canvas Headless — Astro</title>
        </>
      )
    }
  </head>
  <body>
    <DraftBanner />
    <slot />
  </body>
</html>
```

Rationale:

- `page.head` always contains `title` and may contain `meta`, `link` and
  `script` (JSON-LD) entries; canonical links are omitted because the frontend
  owns its public URLs (`modules/canvas_headless/README.md`, "Canvas content
  endpoint"). Rendering it with Unhead produces the complete head markup,
  including charset and viewport tags, which is why the layout inserts it with
  `set:html` and only falls back to static tags when no head was supplied.
- The global stylesheet import moves here from nowhere (fresh Astro has none).
  Astro bundles it into every page that uses the layout. It is the same file
  `canvas.config.json` names, so component previews and pages share it.
- `<DraftBanner />` is described in the next step. It renders nothing unless a
  draft session is active.

### 7. Add the draft session banner

**Tested** (renders and loads its script against the mock; the full session
lifecycle was then verified against a real Drupal site, see
[Real Drupal validation](#real-drupal-validation)). Create
`src/components/DraftBanner.astro`:

```astro
---
import DraftSession from '@drupal-canvas/headless-astro/DraftSession.astro';
---

<DraftSession>
  <div data-draft-session-view="expired" class="draft-banner draft-banner--expired">
    <span><strong>Draft preview session expired.</strong> Showing only content visible to anonymous visitors.</span>
    <span>
      <a data-draft-session-renew-link>Renew session</a>
      <form method="POST" action="/api/disable-draft"><button type="submit">Exit draft mode</button></form>
    </span>
  </div>
  <div data-draft-session-view="active" class="draft-banner draft-banner--active">
    <span><strong>Draft mode is active.</strong> You may be seeing unpublished content.</span>
    <form method="POST" action="/api/disable-draft"><button type="submit">Exit draft mode</button></form>
  </div>
</DraftSession>

<style is:global>
  .draft-banner { display: flex; justify-content: space-between; gap: 1rem; padding: 0.5rem 1rem; font-size: 0.875rem; }
  .draft-banner--active { background: #fcd34d; color: #451a03; }
  .draft-banner--expired { background: #fecaca; color: #450a0a; }
  .draft-banner form { display: inline; }
  .draft-banner button, .draft-banner a { font: inherit; font-weight: 600; text-decoration: underline; background: none; border: 0; padding: 0; cursor: pointer; }
</style>
```

The template's `astro/src/components/DraftBanner.astro` is the same structure
with Tailwind classes. Rationale
(`packages/headless-astro/src/components/DraftSession.astro`):

- `DraftSession.astro` reads the session state server-side and renders the
  `<canvas-draft-session>` custom element from `@drupal-canvas/headless/client`
  with `token-expires-at`, `renew-url`, `editor-origin` and `renew-endpoint`
  attributes. The element owns the state machine: expiry timing, the postMessage
  renewal protocol with the embedding Canvas editor, height reporting to the
  host, delegated navigation, and refresh after auto-save (which it performs
  through Astro's `navigate()`, so a plain document navigation without
  `ClientRouter`).
- Your slot content owns presentation. Children marked
  `data-draft-session-view="active"` show while the session is live and the page
  is standalone; `data-draft-session-view="expired"` children show once it
  expired, also inside the editor iframe. An anchor marked
  `data-draft-session-renew-link` gets its `href` pointed at Drupal's renew
  route. Until the element is defined, all views are hidden by a global style
  the component emits, so first paint shows nothing.
- Exiting draft mode is a `POST` form, not a link, because `GET` links are
  eligible for prefetching and would end the session without a click
  (`packages/headless-astro/src/routes/disable-draft.ts`).
- Astro's `security.checkOrigin` default (`true` in
  `node_modules/astro/dist/core/config/schemas/defaults.js`) rejects form
  `POST`s whose `Origin` does not match the site with a 403 "Cross-site POST
  form submissions are forbidden". Browsers always send `Origin` on form posts,
  so the banner works; tested with and without the header using curl.
- If you mount the renewal route somewhere else (`injectRoutes: false`), pass
  `renewEndpoint` to `DraftSession`.

### 8. Add Canvas components

**Tested.** One folder per component under `componentDir`, holding
`component.yml` and `index.astro`. Example `src/components/hello/component.yml`:

```yaml
name: Hello
machineName: hello
status: true
required:
  - heading
props:
  properties:
    heading:
      title: Heading
      type: string
      examples:
        - Hello from Astro
    tone:
      title: Tone
      type: string
      enum:
        - neutral
        - accent
      meta:enum:
        neutral: Neutral
        accent: Accent
      examples:
        - neutral
slots:
  content:
    title: Content
```

And `src/components/hello/index.astro`:

```astro
---
interface Props {
  heading: string;
  tone?: 'neutral' | 'accent';
}
const { heading, tone = 'neutral' } = Astro.props;
---

<section class:list={['hello p-8', tone === 'accent' && 'bg-primary-600/10']}>
  <h2>{heading}</h2>
  <div class="hello__content"><slot name="content" /></div>
</section>
```

Rationale (`packages/discovery/src/discover.ts`,
`packages/headless/src/component-entry-extensions.ts`,
`packages/headless/src/component-registry.ts`,
`packages/headless-astro/src/components/CanvasElement.astro`, and the
`canvas-headless` skill in `astro/.agents/skills/canvas-headless/SKILL.md`):

- Discovery scans `componentDir` for `component.yml` (or `*.component.yml`) and
  resolves the entry file beside it. Headless discovery accepts the JavaScript
  extensions plus `.astro`, `.vue` and `.svelte`. A metadata file without an
  entry is skipped with a warning; duplicate `machineName`s are reported and the
  first wins in the registry.
- `machineName` is the component's identity in Canvas and the key of the
  generated registry. Drupal renders a placed component as the custom element
  `js-<machineName>` (dots, colons and underscores become hyphens), and the
  renderer maps it back (`packages/headless/src/render.ts`).
- Props arrive as `Astro.props`; rich text props (`contentMediaType: text/html`)
  are rendered with `set:html`; images arrive as objects with `src`, `alt`,
  `width`, `height`. Slots arrive as named Astro slots:
  `<slot name="content" />`. The renderer constructs slot functions at runtime
  because Canvas slot names are only known from the tree.
- Only metadata is registered with Drupal (`type: external`); the app renders
  its own implementation. The metadata endpoint's payload (`version: 1`,
  `components[]` with `machineName`, `name`, `status`, `required`, `props`,
  `slots`, `relativeDirectory`, and `warnings[]`) is what Drupal's
  `ExternalComponentSync` reads
  (`modules/canvas_headless/src/ExternalComponentSync.php`). Components missing
  from a later payload are retained on the Drupal side, never deleted.
- Do not write a manual registry. The Vite plugin generates
  `virtual:@drupal-canvas/headless/components` with one static import per
  discovered component and invalidates it when components are added, removed or
  renamed during `astro dev`.
- The full `component.yml` schema (enums with `meta:enum`, image and entity
  reference shapes, `x-formatting-context`) is documented in the
  `canvas-component-metadata` skill shipped in this template's `.agents/skills/`
  directory and in the Canvas docs. `mocks.json` files are Workbench mocks and
  are unused in Astro projects.
- To reuse the template's 18 components, copy their directories from
  `astro/src/components/` into your `componentDir`. They use Tailwind utilities
  and the `primary-*` theme tokens from `astro/src/styles/global.css`, so they
  need the [Tailwind step](#tailwind-css) and those tokens.

### 9. Ignore build artifacts

**Tested.** Append to `.gitignore`:

```gitignore
# Canvas build artifacts
.canvas/

# every env file except the example
.env.*
!.env.example
```

`.canvas/components.manifest.json` is written on every build. It lives outside
`public/` on purpose: the metadata endpoint that serves it is authenticated, so
the file must not be reachable by another route
(`packages/headless/src/components-endpoint/manifest-read.ts`). Fresh Astro
already ignores `.env` and `.env.production`; the template ignores every
`.env.*` variant.

At this point the migrated copy passed `astro check` (0 errors) and
`astro build`, and the built server answered every injected route as described
in the [verification log](#verification-log).

## Optional steps

### Tailwind CSS

**Tested.** The template styles everything with Tailwind CSS 4 and the
`primary-*` colour tokens its components expect. Astro's integration command
does the install and config edit:

```bash
npx astro add tailwind --yes
```

That installed `tailwindcss` and `@tailwindcss/vite` (4.3.3 at the time) and
added `vite: { plugins: [tailwindcss()] }` to `astro.config.mjs`, exactly the
shape the template uses. It did not touch the existing `src/styles/global.css`,
so replace it with the template's version (`astro/src/styles/global.css`) or
this reduced one:

```css
@import 'tailwindcss';
@source '../components';

@theme {
  --font-sans: Inter, Roboto, 'Helvetica Neue', Arial, sans-serif;
  --color-primary-600: var(--color-blue-600);
  --color-primary-700: var(--color-blue-700);
}

html {
  color: var(--color-slate-950);
}
body {
  margin: 0;
}
```

`@source '../components'` makes Tailwind scan the component directory
explicitly; the template's full file defines the `primary-100` to `primary-900`,
`primary-dark` and `primary-light` tokens its components use. The template also
installs `@tailwindcss/typography` for prose styling; add it with
`npm install -D @tailwindcss/typography` and
`@plugin '@tailwindcss/typography';` in the CSS if you copy components that rely
on it. After the switch, `astro check` and `astro build` passed and the built
page carried the compiled `.p-8` utility and the `--color-primary-600` token.

### Trusting local HTTPS certificates

**Inferred** (the template does it; it was not exercised against HTTPS here).
Node.js does not trust the operating system's certificate store, so a DDEV or
other locally signed Drupal site fails TLS verification. The template's
`src/middleware.ts` calls the SDK helper once at module load:

```ts
import { trustSystemCertificates } from '@drupal-canvas/headless/node';
import { defineMiddleware } from 'astro:middleware';

// Node.js does not trust system certificates by default; local DDEV HTTPS requires them.
trustSystemCertificates();

export const onRequest = defineMiddleware((_context, next) => next());
```

The integration registers its own CSP middleware separately, so this file only
exists to run the certificate call in the server process
(`packages/headless/src/node.ts`). Skip it when Drupal is served over plain HTTP
or a publicly trusted certificate.

### Linting, type checking, and engines

**Inferred.** The template adds:

```json
{
  "engines": { "node": ">=22.19.0 <23 || >=24.5.0" },
  "scripts": {
    "lint": "eslint .",
    "type-check": "astro check",
    "check": "npm run lint && npm run type-check"
  }
}
```

with `eslint`, `eslint-plugin-astro` and `@drupal-canvas/eslint-config` as dev
dependencies and this `eslint.config.js`:

```js
import { recommended as drupalCanvasRecommended } from '@drupal-canvas/eslint-config';
import eslintPluginAstro from 'eslint-plugin-astro';

export default [
  ...drupalCanvasRecommended,
  ...eslintPluginAstro.configs.recommended,
  {
    ignores: ['.astro/**', '.canvas/**', 'dist/**', 'node_modules/**'],
  },
];
```

The migrated copy only added `"type-check": "astro check"`. The `engines` range
mirrors the SDK's; `.nvmrc` at the repository root pins Node 22.

### Canvas CLI

**Inferred.** The template installs `@drupal-canvas/cli` as a dev dependency and
reserves `CANVAS_CLIENT_ID`/`CANVAS_CLIENT_SECRET` in `.env.example` for it. In
a headless codebase the CLI detects the SDK from `package.json` and changes
behaviour: `canvas push` sends metadata only, `canvas pull` excludes external
components (useful once to migrate Drupal-hosted components), `canvas scaffold`
is React-only, and `canvas build`/`canvas validate` report no components
(`astro/.agents/skills/canvas-headless/SKILL.md`, "CLI semantics in a headless
codebase"). The editor synchronizes components from the metadata endpoint on
load, so the CLI is not required for the integration.

### Agent skills

**Inferred.** The template ships Canvas skills under `.agents/skills/` with
`skills-lock.json`, installed with `npx skills add drupal-canvas/skills`
(`/home/hazelnut/workspace/skills/README.md`). They are documentation for AI
coding agents, not runtime code. Fresh Astro instead generated `AGENTS.md` and a
`CLAUDE.md` symlink with Astro-specific guidance; the template removed its
Claude symlinks in a later commit.

### Package manager notes

**Inferred.** The template's `pnpm-workspace.yaml`
(`allowBuilds: esbuild: true`) approves esbuild's post-install script for
pnpm 10. Fresh Astro's `package.json` carries the npm 11 equivalent,
`allowScripts`. Keep whichever matches your package manager. Do not add npm
workspaces or hoist dependencies between templates (`CONTRIBUTING.md`).

## Drupal-side configuration

**Tested** against a local Drupal site (Drupal installed from the sandbox's
project definition with `canvas`, `canvas_headless`, `simple_oauth` and its
generated keypair, served at `http://127.0.0.1:8080`; the migrated app served at
`http://127.0.0.1:4321`). The steps follow `modules/canvas_headless/README.md`
and `docs/user/src/content/docs/headless/setup.mdx`:

1. Enable the `canvas_headless` module. It requires `simple_oauth` (6.1.0 or
   later) with its RSA keypair configured at
   `/admin/config/people/simple_oauth`, `consumers`, and `custom_elements`. The
   module provisions the `canvas_headless` OAuth consumer and scope itself.
2. Grant **Administer Canvas Headless frontends** to roles that manage the
   frontend list and **Access Canvas Headless preview** to editorial roles.
3. Start the app and confirm the contract from a terminal:

   ```bash
   curl -i http://localhost:4321/api/canvas/components
   ```

   A `401` with `www-authenticate: Bearer` and a
   `content-security-policy: frame-ancestors ...` header is the success case; it
   is the exact probe the editor runs.

4. In the Canvas editor, open **Headless frontends** in the icon rail, add the
   app URL (no trailing slash, query or fragment; the config schema in
   `modules/canvas_headless/config/schema/canvas_headless.schema.yml` rejects
   anything else), and wait for the row to show **Ready**. The browser, not the
   Drupal server, performs the probe, so `localhost` works.
5. Open any page in the editor. The editor embeds the first registered app in an
   iframe with an active draft session and, on the same load, fetches the
   metadata endpoint with a freshly minted assertion and registers every
   component it finds. Reload the editor after adding a component.
6. Publish a page and load the same path on the app: it renders as ordinary
   markup, without editor markers.

Two things the local run showed that the docs do not spell out:

- A fresh site's front page is the `/node` listing view. The content API answers
  406 for it (Drupal's `RequestFormatRouteFilter` logs "No route found for the
  specified format. Supported formats: html"), so the app's `/` is a 404 until
  the front page is a Canvas page or another entity route. Any entity path
  (`/node/1`, an alias) resolves.
- Component synchronization runs every time the editor loads with a frontend
  selected, and it needs the **Administer Canvas Headless frontends** permission
  on top of the preview permission.

Browser constraints for the embedded preview: Chromium works over HTTPS and over
plain-HTTP `localhost`; Firefox needs HTTPS (and a third-party cookie exception
if those are blocked); Safari needs CHIPS support (absent in 18.5 through 26.1).

## How the integration works at runtime

All **Inferred** from the SDK sources unless marked otherwise.

### Request lifecycle for a public page

1. The CSP middleware computes the draft state for the request and, after the
   route responds, merges `frame-ancestors` into the response's
   `Content-Security-Policy`. Without `CANVAS_EDITOR_ORIGINS`, the sources are
   `'self'`, the `CANVAS_SITE_URL` origin, and the editor origin from a live
   session; an app-owned `frame-ancestors` stays authoritative
   (`packages/headless-astro/src/middleware.ts`,
   `packages/headless/src/server/csp.ts`). **Tested:** every response carried
   `content-security-policy: frame-ancestors 'self' http://127.0.0.1:8081`.
2. The catch-all route calls `fetchPage(Astro, path)`. With no session the
   request to `/canvas/content-api` is anonymous and returns only what anonymous
   visitors see.
3. `CanvasComponentTree.astro` walks `page.content`. Structural elements
   (`renderless-container`, `drupal-markup`) render their slot children; `js-*`
   elements resolve to the registry; unknown components are logged
   (`[canvas] Canvas component "x" is not registered; omitted subtree`) and
   skipped. **Tested** with a mock tree containing one known and one unknown
   component.
4. In draft mode (`content.canvasDraftMode === true`, set by the SDK only when a
   live session fetched a Canvas-managed route), the renderer additionally emits
   HTML comment markers for regions, components and slots, empty-slot
   placeholders, and imports `@drupal-canvas/headless/preview.css`, which Canvas
   uses for selection and drag-and-drop overlays.

### Draft preview lifecycle

1. The editor loads the iframe at `{frontend}/api/draft?assertion=<jwt>`. The
   route redeems the assertion at `{CANVAS_SITE_URL}/oauth/token` with the RFC
   7523 `jwt-bearer` grant and a fresh PKCE challenge, then sets two cookies,
   `canvas_headless_draft_mode` (the flag) and `canvas_headless_draft_data`
   (path, resource version, subject, renew URL, access token, expiry, PKCE
   verifier), and redirects to the assertion's signed entry path
   (`packages/headless/src/server/flows.ts`,
   `packages/headless-astro/src/adapter.ts`). The cookies are
   `HttpOnly; Secure; Partitioned; SameSite=None` so they survive inside a
   cross-site iframe. **Tested:** the exit route's cleared cookies show exactly
   these attributes, and against the real site the editor's iframe loaded
   `/api/draft?assertion=...`, Drupal issued the access token (visible in its
   `oauth2_token` table), and the editor reported "Draft session active".
2. Every subsequent page render reads the session, sends the user-bound bearer
   token to the content API, and forwards preview context (`language`,
   `viewMode`, `pageVariant`, `excludeAutoSave`) from the URL. **Tested:** the
   preview rendered an unpublished page and, after an edit in the editor, the
   auto-saved heading, while anonymous requests to the same path got 404 and,
   once published, the published heading only.
3. Before the token expires, `<canvas-draft-session>` asks the host for a new
   assertion over postMessage and posts it to `/api/draft/renew`, which answers
   `{tokenExpiresAt}`; a renewal for a different editor is refused with 409.
   **Tested:** a renew call without a session answers 400.
4. Standalone (not embedded), the expired banner's renew link performs a
   top-level navigation through Drupal's renew route, the one request shape that
   carries Drupal's `SameSite=Lax` session cookie cross-site.
5. `POST /api/disable-draft` clears both cookies and answers 303 to `/`.
   **Tested.**
6. `/api/canvas/component-preview?componentId=<id>` renders one component in
   isolation for library thumbnails; it redirects to `/` without a session.
   **Tested** (302 without a session).

Activation without an assertion answers 422 (**Tested**). Failure responses from
Drupal's token endpoint pass their status through; a network failure
answers 502.

### JSON:API access from server code

`getClient(Astro)` returns the shared `drupal-canvas` JSON:API client
(`createJsonApiClient()`), authenticated with the session token while the
session is live and anonymous otherwise. The JSON:API prefix is discovered once
per server from `{CANVAS_SITE_URL}/canvas/api/v0/site-data`, with
`CANVAS_JSONAPI_PREFIX` and then `/jsonapi` as fallbacks
(`packages/headless/src/server/site-data.ts`). Responses use
`DefaultSerializer`: collections are arrays with attributes flattened onto each
resource. The template's `src/lib/content.ts` shows the pattern for articles and
Canvas pages and links them through the catch-all route; nothing in the template
calls it. `fetchEntity(Astro, { type, id, viewMode })` renders one entity
through a content template without route or head data.

`page.context` carries the page and site context Drupal generated (title,
breadcrumbs, primary entity, branding). In Astro it is plain data on the
`fetchPage()` result for your own use; no React providers are involved.

### JSON:API proxy for browser code

The integration mounts `ALL /api/canvas/jsonapi/[...path]`. Browser clients
built from `getJsonApiRuntimeConfig(Astro)` (serialized into the page with
`serializeJsonForHtml()`) send requests through it, and the proxy adds the
session's token, refuses anything outside the backend's JSON:API prefix and
Decoupled Router endpoint, never forwards browser cookies or auth headers, and
rejects state-changing requests that lack a same-origin `Origin` or
`Sec-Fetch-Site` signal (`packages/headless/src/server/jsonapi-proxy.ts`).
**Tested:** `GET /api/canvas/jsonapi/jsonapi/node/article` was forwarded to the
mock backend's `/jsonapi/node/article` and returned with
`cache-control: no-store` and `vary: Cookie`;
`GET /api/canvas/jsonapi/node/article` (prefix missing) answered
`404 endpoint_not_allowed`; a cross-site `POST` answered 403. The path after the
proxy base is the upstream path including the `jsonapi` prefix. An expired
session answers `401 draft_session_expired`, never public content.

### Component metadata endpoint

`GET /api/canvas/components` requires a Drupal-minted preview assertion as a
Bearer token and verifies it by redeeming it at Drupal's token endpoint (proof
by redemption). The authenticated 200 is CORS-readable only by the editor origin
carried in the assertion's `renewUrl` claim. Every response is
`Cache-Control: no-store`. In `astro dev` each request scans the codebase live;
in a build the manifest inlined into the server bundle is served, and a missing
manifest is a 500, never an empty registry
(`packages/headless/src/components-endpoint/handler.ts`,
`packages/headless-astro/src/routes/components.ts`). **Tested:** 401 without a
token in both dev and production; `OPTIONS` preflight answers 204 with
`Access-Control-Allow-Headers: Authorization`; the built bundle
`dist/server/chunks/components_*.mjs` contains the manifest JSON.

## Build and deployment

**Tested** locally with the standalone Node adapter:

```bash
npm run build
CANVAS_SITE_URL=https://your-drupal-site.example.com HOST=0.0.0.0 PORT=4321 node ./dist/server/entry.mjs
```

- `astro build` first writes `.canvas/components.manifest.json` and logs
  `Wrote the component manifest: N component(s), M warning(s).`; the manifest is
  then inlined into the server bundle, so neither the component sources nor the
  `.canvas/` file are read at runtime.
- `dist/` is not self-contained. The Node adapter leaves the app's production
  dependencies external: `dist/server/entry.mjs` and its chunks import
  `unstorage`, `unhead/server`, `yaml`, `zod`, `send`, `@oslojs/encoding` and
  others from `node_modules`. Tested: copying `dist/` elsewhere and running it
  fails with `ERR_MODULE_NOT_FOUND: Cannot find package '@oslojs/encoding'`.
  Deploy `dist/` together with `package.json`, `package-lock.json` and a
  production install (`npm ci --omit=dev`).
- `CANVAS_SITE_URL` (and any optional `CANVAS_*` variable) must be present in
  the server process environment. `.env` files are not read by the built server.
  `CANVAS_JSONAPI_PROXY_PATH` must carry the same value at build and run time
  (step 4).
- `HOST` and `PORT` are the adapter's runtime overrides; `SERVER_CERT_PATH` and
  `SERVER_KEY_PATH` enable HTTPS
  (<https://docs.astro.build/en/guides/integrations-guide/node/>).
- The metadata endpoint has to be reachable by the editors' browsers at the URL
  registered in Drupal, and the Drupal origin has to be allowed in
  `frame-ancestors`. If a CDN or reverse proxy adds its own
  `Content-Security-Policy`, remember that multiple policies intersect.
- `npm run preview` (`astro preview`) serves the same build through the Node
  adapter (**Tested:** page and metadata endpoint answered as in production).

## Verification log

All commands ran on 2026-09-26 in `/home/hazelnut/workspace/astro-comparison/`
(baseline untouched; migrated copy at `migrated/`; scaffold output in
`scaffold.log`; browser screenshot in `migrated-home.png`; real-Drupal
screenshots in `evidence/`). Scratch copies used for checks lived in the session
scratchpad and are not part of the deliverable.

| Check                            | Command                                                                                                                              | Result                                                                                                                                               |
| -------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------- |
| Scaffold                         | `npm create astro@latest baseline -- --yes --no-git --install`                                                                       | exit 0, `basics` template, astro 7.3.5, 15 s                                                                                                         |
| Baseline builds                  | `npm run build` (scratch copy of baseline)                                                                                           | static build, `dist/index.html`                                                                                                                      |
| Template check with its lockfile | `npm ci && npm run check && npm run build` (scratch copy of `astro/`, astro 7.1.6)                                                   | lint 0 problems, `astro check` 0 errors, build OK                                                                                                    |
| SDK install                      | `npm install @astrojs/node @drupal-canvas/headless-astro @drupal-canvas/headless unhead`; `npm install -D @astrojs/check typescript` | versions in the table above                                                                                                                          |
| Type check before redirect fix   | `npx astro check`                                                                                                                    | 1 error: `ts(2345)` on `Astro.redirect(..., statusCode)`                                                                                             |
| Type check after fix             | `npx astro check`                                                                                                                    | 0 errors, 0 warnings                                                                                                                                 |
| Build                            | `npm run build`                                                                                                                      | manifest with 1 component, 0 warnings; `dist/server/entry.mjs`                                                                                       |
| Built server without env         | `env -u CANVAS_SITE_URL node dist/server/entry.mjs`                                                                                  | `GET /api/canvas/components` → 500 `configuration_error`                                                                                             |
| Built server with env            | `CANVAS_SITE_URL=http://127.0.0.1:8081 node dist/server/entry.mjs`                                                                   | see rows below                                                                                                                                       |
| Metadata endpoint                | `curl -i /api/canvas/components`                                                                                                     | 401 `missing_assertion`, `www-authenticate: Bearer`, `cache-control: no-store`, CSP header                                                           |
| Preflight                        | `curl -X OPTIONS -H 'Origin: http://127.0.0.1:8081' ...`                                                                             | 204 with CORS headers                                                                                                                                |
| Page render                      | `curl /` against the mock backend                                                                                                    | 200, `<title>Mock front page</title>`, meta description, `hello` component rendered with slot markup; unknown component logged and skipped           |
| Redirect                         | `curl -i /old`                                                                                                                       | 301, `location: /`                                                                                                                                   |
| Not found                        | `curl -i /nope`                                                                                                                      | 404 with the layout's not-found branch                                                                                                               |
| Draft activation                 | `curl -i /api/draft`                                                                                                                 | 422 "Missing preview assertion"                                                                                                                      |
| Renewal                          | `curl -X POST /api/draft/renew -d '{"assertion":"x"}'`                                                                               | 400 (no session)                                                                                                                                     |
| Exit draft                       | `curl -X POST /api/disable-draft` without `Origin` / with same-origin `Origin`                                                       | 403 (Astro CSRF check) / 303 to `/`, both cookies cleared with `Partitioned; SameSite=None`                                                          |
| JSON:API proxy                   | `curl /api/canvas/jsonapi/jsonapi/node/article`                                                                                      | 200, forwarded to the mock; missing prefix → 404 `endpoint_not_allowed`; cross-site POST → 403                                                       |
| Component preview                | `curl -i '/api/canvas/component-preview?componentId=hello'`                                                                          | 302 to `/` (no session)                                                                                                                              |
| Dev server with `.env` only      | `env -u CANVAS_SITE_URL npx astro dev --port 4322`                                                                                   | 401 on the metadata endpoint, CORS headers on a cross-origin GET, page renders                                                                       |
| Tailwind                         | `npx astro add tailwind --yes`, CSS swap, `astro check`, `npm run build`                                                             | config edited as expected; 0 errors; built CSS contains `.p-8` and `--color-primary-600`                                                             |
| Preview                          | `CANVAS_SITE_URL=... npx astro preview --port 4323`                                                                                  | 200 on `/` with the mock title, 401 on the metadata endpoint                                                                                         |
| Browser                          | agent-browser on the built page                                                                                                      | screenshot taken, no console errors, `customElements.get('canvas-draft-session')` defined, computed padding 32px and tinted background from Tailwind |

The "mock backend" is the Node script
`/home/hazelnut/workspace/astro-comparison/mock-drupal.mjs` that answers
`/canvas/api/v0/site-data`, `/canvas/content-api` (a fixed tree for `/`, a
redirect for `/old`, 404 otherwise) and `/jsonapi/node/article`. It proves the
Astro side of the contract, not Drupal's.

### Real Drupal validation

A second pass on 2026-09-26 installed Drupal locally (`composer install`, the
sandbox's `site-install`, which enables `canvas`, `canvas_headless`,
`simple_oauth` and generates the RSA keypair, then the PHP built-in server on
`http://127.0.0.1:8080`) and ran the migrated app's production build on
`0.0.0.0:4321` with `CANVAS_SITE_URL=http://127.0.0.1:8080`. All browser steps
used a Chromium session driven by agent-browser; screenshots are in
`/home/hazelnut/workspace/astro-comparison/evidence/`.

| Check                                   | How                                                                                              | Result                                                                                                                                                                                                                                                                                 |
| --------------------------------------- | ------------------------------------------------------------------------------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Registration                            | Canvas editor, Headless frontends, Add frontend `http://127.0.0.1:4321`                          | Row shows **Ready**; `canvas_headless.settings` gained the frontend (`e2e-frontends.png`)                                                                                                                                                                                              |
| Component sync                          | Open the editor for an entity                                                                    | "Component sync completed: Created 1"; `canvas.js_component.hello` exists with `type: external`; the frontend's `components` list holds `js.hello`                                                                                                                                     |
| Metadata endpoint with a real assertion | Same load                                                                                        | Drupal's `oauth2_token` table gained a token for the `canvas_headless` consumer, the proof-by-redemption of the sync fetch                                                                                                                                                             |
| Assertion issuance and redemption       | Same load                                                                                        | Iframe `src` is `http://127.0.0.1:4321/api/draft?assertion=<jwt>`; a second token appears; status line reads "Draft session active — renews automatically around …"                                                                                                                    |
| Draft preview of unpublished content    | Canvas page created unpublished with one `hello` component at `/hello-e2e`, opened in the editor | Preview renders the component with the accent tint, the empty **Content** slot placeholder and the selection outline (`e2e-editor-page1-draft.png`); anonymous `GET /hello-e2e` on the app is 404 and the content API answers 403                                                      |
| Draft edit and auto-save refresh        | Change the Heading prop in the editor                                                            | Preview iframe shows the new heading without a reload; "Review 1 change" appears (`e2e-editor-page1-edited.png`)                                                                                                                                                                       |
| Publish                                 | "Publish 1 selected", then the page set published                                                | Anonymous `GET /hello-e2e` is 200 with the published heading, no `<!-- canvas-… -->` markers and no `<canvas-draft-session>` element (`e2e-public-hello-e2e.png`, `public-hello.html`)                                                                                                 |
| Draft versus published                  | A further edit left unpublished                                                                  | Editor preview shows "Second draft, not yet published"; anonymous page keeps the published heading (`e2e-editor-second-draft.png`)                                                                                                                                                     |
| Standalone draft mode                   | Open `http://127.0.0.1:4321/hello-e2e` in a top-level tab of the same browser                    | Draft content, editor markers and the yellow "Draft mode is active" banner render standalone (`e2e-standalone-draft.png`). This works because `127.0.0.1:8080` and `127.0.0.1:4321` are the same site for cookie purposes; a cross-site deployment keeps the session inside the iframe |
| Node article without a content template | Open `/canvas/editor/node/1`                                                                     | Preview iframe starts a session, but Canvas itself reports "For now Canvas only works if the entity is a canvas_page", as documented for content without a content template                                                                                                            |
| Front page                              | `GET /` on the app                                                                               | 404, because the site's front page is the `/node` view and the content API answers 406 for it                                                                                                                                                                                          |

Not verified even in this pass:

- Session renewal at expiry (the token lifetime was longer than the session),
  the standalone renew link, and the isolated component preview thumbnail.
- `trustSystemCertificates()` against a locally signed HTTPS site (the site ran
  over plain HTTP).
- Firefox and Safari behaviour.
- A cross-site deployment (different hosts for Drupal and the app), which is
  where the partitioned cookies matter. A same-site setup like this one does not
  exercise them.

## Known differences and pitfalls

- **`astro check` and the language server:** narrow the redirect status before
  passing it to `Astro.redirect()` (step 5). The error only surfaces with
  `@astrojs/language-server` 2.17.1 or later, which a fresh install resolves;
  the template lockfile's 2.16.13 misses it. Astro 7.1.6 and 7.3.5 behave the
  same otherwise: the template compiled unchanged on 7.3.5.
- **TypeScript 6:** a fresh `npm install -D typescript` picks 6.x; `astro check`
  0.9.10 worked with it in this exercise. Pin `^5.9.3` to match the template if
  you want identical tooling.
- **`dist/` needs `node_modules`:** the built server imports its production
  dependencies; deploy with `npm ci --omit=dev`.
- **Front page:** a stock site's `/` is the `/node` view, which the content API
  cannot serve (406); make the front page a Canvas page or route `/` yourself.
- **`.env` in production:** only `astro dev` and `astro build` read it. The
  built server needs the variables in its environment.
- **Missing `globalCssPath` file breaks the build** because the injected
  component preview page imports it through a virtual module.
- **Default index page:** delete `src/pages/index.astro`, otherwise `/` never
  reaches Drupal.
- **Form posts need a same-origin `Origin`:** Astro's default CSRF check rejects
  cross-site form posts; browsers satisfy it, curl does not.
- **Proxy path shape:** the JSON:API proxy expects the upstream path, including
  the `jsonapi` prefix, after `/api/canvas/jsonapi`. If you change
  `CANVAS_JSONAPI_PROXY_PATH`, set it identically for the build and the running
  server, and rebuild.
- **Unreachable Drupal:** `fetchPage()` throws on network errors; the catch-all
  route then renders Astro's 500 page. Catch it if you prefer a branded error.
- **Tailwind preflight** resets heading styles; the template's components set
  their own sizes.
- **Dev server host checks:** the integration relaxes Astro's
  `security.allowedDomains` and disables Vite's CORS middleware in `astro dev`
  so the editor can reach the metadata endpoint; production host validation is
  unchanged.

## Daytona-only workarounds (not applied)

The Canvas checkout's `HAZELNUT.md` ("Headless Canvas previews on Daytona")
describes a sandbox-local workaround for previewing through Daytona's signed
preview proxy: an extra `X-Daytona-Skip-Preview-Warning` request header in the
editor's metadata fetch, allowing that header in the SDK handler's
`Access-Control-Allow-Headers`, and suppressing the app's
`Access-Control-Allow-Origin` for the exact Drupal origin when the proxy adds
its own. None of that is part of the portable integration described above, none
of it belongs in a project, and it must never be committed. Apply it only when
previewing through Daytona and only after reproducing the duplicate-header
problem with real preflight and GET evidence, following that runbook.

During the real-Drupal pass the problem was reproduced through the sandbox's
signed preview URLs: the proxy answered the CORS preflight itself and added a
second `Access-Control-Allow-Origin` to the app's GET, so the browser's fetch
from the Drupal origin failed and the frontend row showed **Setup needed**. Only
step 4 of the runbook was applied, and only to the scratch app's installed copy
of the handler
(`node_modules/@drupal-canvas/headless/dist/components-endpoint/handler.js`,
followed by a rebuild, since the Astro integration bundles the SDK into
`dist/`): the app's own `Access-Control-Allow-Origin` is suppressed for that one
Drupal origin. After that the row showed **Ready**, the sync registered
`js.hello` for the proxied frontend, and the editor previewed the page through
the proxied app with an active draft session. The Canvas UI and core-package
edits of step 3 were not needed for this proxy and were not made. The signed
hostnames are deliberately absent from this guide.
