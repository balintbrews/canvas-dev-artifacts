# Minimal Astro to Canvas Headless recipe

The smallest set of changes that turns a fresh `npm create astro@latest` app
into a Canvas Headless frontend: Drupal-routed pages, draft preview for the
Canvas editor, and one component Canvas can place. It assumes a Drupal site with
the `canvas_headless` module already set up; registering the app there and
syncing components are covered in the
[full guide](./astro-to-canvas-headless-guide.md#drupal-side-configuration).
Every snippet below was applied verbatim to a fresh Astro 7 app and passed
`astro check` and `astro build`.

## 1. Install

```bash
npm install @astrojs/node @drupal-canvas/headless-astro unhead
```

`@astrojs/node` gives per-request rendering, which draft preview needs. `unhead`
renders the document head Drupal returns; it is the one convenience dependency
here, replaceable by hand-written `<title>`/`<meta>` tags.

## 2. Configure Astro

Replace `astro.config.mjs`:

```js
import node from '@astrojs/node';
import canvas from '@drupal-canvas/headless-astro/integration';
import { defineConfig } from 'astro/config';

export default defineConfig({
  output: 'server',
  adapter: node({ mode: 'standalone' }),
  integrations: [canvas()],
});
```

`canvas()` injects every route Canvas needs (`/api/draft`, `/api/draft/renew`,
`/api/disable-draft`, `/api/canvas/components`, `/api/canvas/jsonapi/*`,
`/api/canvas/component-preview`), the `frame-ancestors` CSP middleware, the
generated component registry and the build-time component manifest. No
`canvas.config.json` is needed when the defaults fit: components in
`src/components`, global CSS at `src/global.css`.

## 3. Environment and ignores

```bash
printf 'CANVAS_SITE_URL=https://your-drupal-site.example.com\n' > .env
printf '\n# Canvas build artifacts\n.canvas/\n' >> .gitignore
```

`CANVAS_SITE_URL` is the whole configuration; there is no client secret. The
built server reads it from the process environment, not from `.env`.

## 4. Global CSS

The integration imports the global stylesheet for component previews, so the
file must exist even if it is nearly empty:

```bash
mkdir -p src/components
printf 'body { margin: 0; font-family: system-ui, sans-serif; }\n' > src/global.css
```

## 5. Replace the starter page with a catch-all route

```bash
rm src/pages/index.astro src/components/Welcome.astro
```

`index.astro` would shadow `/`, which Drupal must serve. Create
`src/pages/[...slug].astro`:

```astro
---
import CanvasComponentTree from '@drupal-canvas/headless-astro/CanvasComponentTree.astro';
import DraftSession from '@drupal-canvas/headless-astro/DraftSession.astro';
import { fetchPage, isPageRedirect } from '@drupal-canvas/headless-astro';
import { createHead } from 'unhead/server';
import '../global.css';

import type { PageHead } from '@drupal-canvas/headless-astro';

const path = `/${(Astro.params.slug ?? '').split('/').map(encodeURIComponent).join('/')}`;
const page = await fetchPage(Astro, path);

if (page && isPageRedirect(page)) {
  const status = page.redirect.statusCode;
  return Astro.redirect(
    page.redirect.url,
    status === 301 || status === 302 || status === 303 || status === 307 || status === 308 ? status : 302,
  );
}
if (!page) {
  Astro.response.status = 404;
}

const head = createHead<Omit<PageHead, 'title'> & { title?: string }>();
head.push(page?.head ?? { title: 'Not found' });
const { headTags } = head.render();
---

<html lang="en">
  <head>
    <Fragment set:html={headTags} />
  </head>
  <body>
    <DraftSession>
      <div data-draft-session-view="active">
        Draft mode is active.
        <form method="POST" action="/api/disable-draft"><button>Exit draft mode</button></form>
      </div>
      <div data-draft-session-view="expired">
        Draft session expired. <a data-draft-session-renew-link>Renew session</a>
      </div>
    </DraftSession>
    {page ? <CanvasComponentTree tree={page.content} /> : <h1>Not found</h1>}
  </body>
</html>
```

What this does:

- `fetchPage(Astro, path)` resolves the path through Drupal's routing with the
  editor's draft session when one is active, and returns `null` for 403/404.
- `Astro.redirect()` only accepts redirect status codes, hence the narrowing.
- `page.head` is Unhead input (`title`, optional `meta`, `link`, `script`);
  Unhead's render also adds the charset and viewport tags.
- `DraftSession` renders nothing without a session. With one, it runs the
  renewal protocol with the editor and shows the `active` view standalone or the
  `expired` view anywhere. The exit control is a `POST` form on purpose: a `GET`
  link could be prefetched and end the session.
- `CanvasComponentTree` renders Drupal's component tree with the generated
  registry and emits the editor markers in draft mode.

## 6. Add one component

`src/components/hello/component.yml`:

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
slots:
  content:
    title: Content
```

`src/components/hello/index.astro`:

```astro
---
interface Props {
  heading: string;
}
const { heading } = Astro.props;
---

<section>
  <h2>{heading}</h2>
  <slot name="content" />
</section>
```

The folder name is free; `machineName` is the component's identity in Canvas and
the registry key. Props arrive as `Astro.props`, slots as named Astro slots.
Discovery accepts `.astro`, `.vue`, `.svelte` and JavaScript entries beside a
`component.yml`. Rich text props are rendered with `set:html`.

## 7. Run

```bash
npm run dev        # http://localhost:4321; /api/canvas/components answers 401
npm run build && CANVAS_SITE_URL=https://your-drupal-site.example.com node dist/server/entry.mjs
```

The 401 without a Bearer assertion is the success signal Canvas probes for.
Register `http://localhost:4321` under **Headless frontends** in the Canvas
editor; components sync when the editor loads. The build needs `node_modules`
next to `dist/` at run time (`npm ci --omit=dev` on the server).

## Left out on purpose

- Tailwind, the template's 18 example components, the styled banner, ESLint, the
  Canvas CLI and agent skills: none are required by the integration.
- `trustSystemCertificates()` from `@drupal-canvas/headless/node` (add it in
  `src/middleware.ts` for a locally signed HTTPS site such as DDEV).
- A stock Drupal front page (`/node` view) is not servable by the content API,
  so `/` answers 404 until the front page is a Canvas page.
- Server-side JSON:API queries (`getClient(Astro)`), `fetchEntity()`, and
  `page.context` are available but unused here; see the
  [full guide](./astro-to-canvas-headless-guide.md#how-the-integration-works-at-runtime).
