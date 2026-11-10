# #3592072 Implementation plan: per-request site selection in the Headless SDK

Work item: https://git.drupalcode.org/project/canvas/-/work_items/3592072
("Support multiple sites from a single Canvas Headless codebase").

This plan was agreed in a design interview on 2026-11-10. It is the hand-over
for an implementation session. Every decision below is settled; do not reopen
one without new facts. Paths are relative to the Canvas checkout
(`~/workspace/canvas`, `$CANVAS_ROOT`) unless stated otherwise. Line numbers
are from commit `c37da212` on `1.x` and may drift.

## 1. Outcome

A headless app can select the Drupal site per request. `CANVAS_SITE_URL`
remains the default. Every SDK operation follows the selected site: page and
entity fetches, JSON:API clients, the same-origin JSON:API proxy, the runtime
config handed to the browser, preview-token exchange and renewal, the
component-metadata endpoint, and component previews. Draft sessions are bound
to the site that issued them.

The config tier is renamed from `Draft*` to `Site*`, as a breaking change
without aliases.

### Settled decisions

| Topic                       | Decision                                                                                                                                                                                                                                                                           |
| --------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Scope                       | SDK only. No Drupal (PHP) changes. Drupal-side domain language negotiation stays unsupported and the multilingual guide says so without an issue link. No new work item, no ADR. Work item 3592073 (custom request headers) stays separate.                                        |
| What a "site" is            | A normalized base URL. Distinct Drupal installs and multiple domains of one install are treated the same.                                                                                                                                                                         |
| Rename (breaking, no alias) | `DraftConfig` → `SiteConfig`, `resolveDraftConfig` → `resolveSiteConfig`, adapter `getDraftConfig` → `getSiteConfig`. Server-tier names (`createDraftServer`, `DraftServer`, `DraftServerOptions`, `DraftServerAdapter`) stay. The Angular template in `headless-templates` is updated by the user, not here. |
| Provider input              | `SiteConfigContext { url, getHeader, getCookie }`, built by the SDK from the adapter. `getHeader` is a new optional adapter method implemented in all five adapters.                                                                                                                 |
| Provider output             | `Partial<SiteConfig> \| undefined`, sync or async, merged over the environment. `undefined` = default site. A `baseUrl` different from `CANVAS_SITE_URL` discards env `CANVAS_JSONAPI_PREFIX`, `CANVAS_JSONAPI_URL`, `CANVAS_JSONAPI_SITE_URL` for that request. `jsonApiProxyPath` always comes from env unless the provider sets it. |
| Registration                | Global `setSiteConfigProvider(provider \| null)` in `@drupal-canvas/headless/server`, called once at app startup. Resolution order per call: explicit `options.config` → registered provider → environment.                                                                          |
| Validation                  | `resolveSiteConfig` validates and normalizes `baseUrl`: `http:`/`https:` only, no credentials, query or fragment; trailing slash stripped; install sub-paths allowed; IP literals and ports allowed. Applies to the env value too.                                                   |
| Discovery cache             | The JSON:API prefix discovered from `/canvas/api/v0/site-data` is cached per normalized site base URL, not once per server instance.                                                                                                                                              |
| Session binding             | Draft cookie gains `site` (normalized base URL of the issuing site). `getDraftData()` returns `null` when the field is missing or differs from the request's resolved site. Old cookies (no field) count as a mismatch.                                                           |
| Not changed                 | CSP `frame-ancestors` resolver; no allowlist of sites beyond "the provider is server code"; no declarative host map.                                                                                                                                                              |
| Verification                | Vitest for the logic; one manual run against two local Drupal multisite installs with a cookie-selected provider on one frontend host, written up in the MR description.                                                                                                           |
| Docs and release            | Headless README, setup guide, adapter READMEs, multilingual guide. `minor` changesets for core and every adapter, each starting with a "Breaking:" line listing the renames.                                                                                                        |

## 2. Public API after the change

In `@drupal-canvas/headless/server`:

```ts
export interface SiteConfig {
  baseUrl: string; // normalized: origin + install path, no trailing slash
  apiPrefix?: string;
  jsonApiUrl?: string;
  jsonApiSiteUrl?: string;
  jsonApiProxyPath?: string;
}

export function resolveSiteConfig(overrides?: Partial<SiteConfig>): SiteConfig;

export interface SiteConfigContext {
  /** Current request URL, or null when the adapter cannot supply one. */
  url: string | null;
  /** A request header, or null when absent or when the adapter cannot read headers. */
  getHeader(name: string): Promise<string | null>;
  getCookie(name: string): Promise<string | null>;
}

export type SiteConfigProvider = (
  context: SiteConfigContext,
) => Partial<SiteConfig> | undefined | Promise<Partial<SiteConfig> | undefined>;

/** Registers the provider every adapter-created server consults. `null` clears it. */
export function setSiteConfigProvider(provider: SiteConfigProvider | null): void;

export interface DraftServerAdapter {
  // existing members unchanged, plus:
  /** Reads a request header; null when absent. Optional for older adapters. */
  getHeader?(name: string): Promise<string | null>;
}

export interface DraftServerOptions {
  adapter: DraftServerAdapter;
  /** Static overrides, or a provider. When set, the global provider is not consulted. */
  config?: Partial<SiteConfig> | SiteConfigProvider;
  fetchImpl?: typeof fetch;
}

export interface DraftServer {
  // `getConfig` becomes async because the provider may be async:
  getConfig(): Promise<SiteConfig>;
  // everything else unchanged
}
```

Draft cookie contract (`DraftData`, isomorphic entry `@drupal-canvas/headless`):
one new required field `site: string`.

App-side usage to document (neutral multi-backend example, not a language one):

```ts
// canvas-sites.ts
import type { SiteConfigProvider } from '@drupal-canvas/headless/server';

const SITES: Record<string, string> = {
  'brand-a.example': 'https://cms.brand-a.example',
  'brand-b.example': 'https://cms.brand-b.example',
};

export const siteConfigProvider: SiteConfigProvider = async ({ getHeader }) => {
  const host =
    (await getHeader('x-forwarded-host')) ?? (await getHeader('host'));
  const baseUrl = host && SITES[host.split(':')[0]];
  return baseUrl ? { baseUrl } : undefined; // undefined → CANVAS_SITE_URL
};
```

Registration per framework (one line each, documented in each adapter README):

- Next.js: `instrumentation.ts` `register()` (the Next template already has
  this file for `trustSystemCertificates()`).
- Nuxt: a Nitro server plugin in `server/plugins/`.
- Astro: module scope of `src/middleware.ts` (runs once per server start).
- TanStack Start: module scope of the server entry.
- Angular: `src/server.ts` before creating the handler, or keep passing
  `{ config: siteConfigProvider }` to `createCanvasHandler()` which already
  accepts it.

## 3. Core package: `packages/headless`

### 3.1 `src/server/config.ts`

- Rename `DraftConfig` → `SiteConfig`, `resolveDraftConfig` → `resolveSiteConfig`.
  Update JSDoc (no "draft" wording for the config; it describes the Drupal
  site and the app's proxy path).
- Add a `normalizeSiteBaseUrl(value: string): string` helper (exported; the
  cookie comparison and tests use it). Implementation: `new URL(value)` (throws
  on relative input → rethrow as a descriptive `Error`), require
  `protocol === 'http:' || 'https:'`, reject `username`/`password`,
  reject non-empty `search`/`hash`, return `normalizeBaseUrl(value)` from
  `drupal-canvas/jsonapi-client` (`origin + pathname` without trailing
  slashes; it already strips search/hash, so the rejection must happen before
  calling it). Error messages name the offending part, e.g.
  `"CANVAS_SITE_URL must be an absolute http(s) URL without credentials, query or fragment; got …"`.
  When the value came from a provider rather than env, say "site base URL"
  instead of `CANVAS_SITE_URL` (pass a `source` label or branch on
  `overrides.baseUrl !== undefined`).
- Merge and discard rule in `resolveSiteConfig(overrides)`:
  1. `envBase = process.env.CANVAS_SITE_URL`; `base = overrides.baseUrl ?? envBase`; throw the existing `CANVAS_SITE_URL must be set. See .env.example.` when neither exists.
  2. `baseUrl = normalizeSiteBaseUrl(base)`.
  3. `siteOverridden = overrides.baseUrl !== undefined && (envBase === undefined || normalizeSiteBaseUrl(envBase) !== baseUrl)`. Guard the env normalization with try/catch: an invalid env value while a provider overrides the site must not throw (treat as overridden).
  4. `apiPrefix`, `jsonApiUrl`, `jsonApiSiteUrl`: `overrides.x ?? (siteOverridden ? undefined : process.env.CANVAS_…)`.
  5. `jsonApiProxyPath`: unchanged logic (override ?? env ?? default; site-relative check).
  6. Keep the `resolveJsonApiBase` validation of `jsonApiUrl` against `baseUrl`.
- Keep `DEFAULT_JSONAPI_PROXY_PATH`.

### 3.2 New `src/server/site-config-provider.ts`

```ts
let registered: SiteConfigProvider | null = null;
export function setSiteConfigProvider(p: SiteConfigProvider | null): void { registered = p; }
export function getSiteConfigProvider(): SiteConfigProvider | null { return registered; }
```

Also define `SiteConfigContext`, `SiteConfigProvider` here (or in
`config.ts`; keep the types next to `SiteConfig` if that reads better) and a
helper used by servers and by the components endpoint:

```ts
export function createSiteConfigContext(source: {
  getRequestUrl?(): Promise<string | null>;
  getHeader?(name: string): Promise<string | null>;
  getCookie(name: string): Promise<string | null>;
}): Promise<SiteConfigContext>;
// url = (await source.getRequestUrl?.()) ?? null
// getHeader = source.getHeader ?? (async () => null)
```

and `createSiteConfigContextFromRequest(request: Request)` for handlers that
only have a web `Request` (components endpoint): `url = request.url`,
`getHeader = name => request.headers.get(name)`, `getCookie` parses the
`cookie` header (first occurrence wins, decodeURIComponent in try/catch; the
Angular adapter in `headless-angular/server/index.ts:42-53` has this logic,
move or copy it).

### 3.3 `src/server/adapter.ts`

Add `getHeader?(name: string): Promise<string | null>` with JSDoc explaining
it feeds `SiteConfigContext` and that `host`/`x-forwarded-host` based
selection needs it.

### 3.4 `src/server/flows.ts`

- `DraftServerOptions.config` type → `Partial<SiteConfig> | SiteConfigProvider`
  (lines 157-169).
- `getConfig` (277-278) becomes async:

  ```ts
  const getContext = () => createSiteConfigContext(adapter);
  const getConfig = async (): Promise<SiteConfig> => {
    if (typeof config === 'function') return resolveSiteConfig(await config(await getContext()));
    if (config) return resolveSiteConfig(config);
    const provider = getSiteConfigProvider();
    return resolveSiteConfig(provider ? await provider(await getContext()) : undefined);
  };
  ```

  Every `getConfig()` call site gets `await`: 333, 451-456, 517, 527, 543.
  `DraftServer.getConfig` (202) becomes `Promise<SiteConfig>`.

- `getDraftData` (280-285): after parsing, bind to the site:

  ```ts
  const draftData = parseDraftData(await adapter.getCookie(DRAFT_DATA_COOKIE_NAME));
  if (!draftData) return null;
  const { baseUrl } = await getConfig();
  return draftData.site === baseUrl ? draftData : null;
  ```

  Resolve config only when a cookie parsed, so requests without a session
  never need site configuration here (keeps `finalize`/CSP paths in
  Astro/Nuxt/Angular working without extra requirements).

- `redeemAssertion` (92-155): add `site: config.baseUrl` to the built
  `draftData` (141-154). `config` is already a parameter.
- Discovery cache (315-321): replace the single memo with a map keyed by site:

  ```ts
  const apiPrefixResolvers = new Map<string, ReturnType<typeof createApiPrefixResolver>>();
  const resolveClientConfig = async (): Promise<SiteConfig> => {
    const clientConfig = await getConfig();
    let resolve = apiPrefixResolvers.get(clientConfig.baseUrl);
    if (!resolve) { resolve = createApiPrefixResolver(clientConfig, fetchImpl); apiPrefixResolvers.set(clientConfig.baseUrl, resolve); }
    const apiPrefix = await resolve();
    return { ...clientConfig, ...(apiPrefix && { apiPrefix }) };
  };
  ```

  Note: a resolver created from a config with `apiPrefix` fallback uses the
  first config's fallback; since the fallback is per site and the key is the
  site, that is correct.

- `handleJsonApiProxy` (497-502) and `getJsonApiRuntimeConfig` (490-495)
  need no change beyond the async `getConfig` they already go through.
- `renewDraftSession`: with a mismatched cookie `existingSession` is now
  `null`, so it returns the existing 400 "no session" response; document in
  the JSDoc that a session from another site counts as no session.

### 3.5 `src/server/site-data.ts`

`createApiPrefixResolver(config: SiteConfig, …)` — rename the type only.
Behavior unchanged (one fetch per resolver, cached forever).

### 3.6 `src/draft-data.ts` (isomorphic)

- `DraftData`: add `site: string` with JSDoc "Normalized base URL of the
  site that issued `accessToken`; sessions are only presented to that site."
- `parseDraftData` (105-129): require `typeof data.site === 'string'`; old
  cookies without it parse to `null`.
- `sessionData` (132-143): include `site`.
- `serializeDraftData`: no change (uses `sessionData`).

### 3.7 `src/components-endpoint/handler.ts`

- Option `config?: (context: SiteConfigContext) => Pick<SiteConfig,'baseUrl'> | Promise<Pick<SiteConfig,'baseUrl'>>` (line 16). Keep accepting a zero-arg function (the context is simply ignored by such callers).
- Default (76): `async (context) => resolveSiteConfig(await getSiteConfigProvider()?.(context))`.
- In `GET` (115-125): build `createSiteConfigContextFromRequest(request)` and `await getConfig(context)` inside the existing try/catch (500 `configuration_error`).

### 3.8 `src/server/jsonapi-proxy.ts`

`JsonApiProxyOptions.getConfig` already returns a Promise of the config
subset; only the type name changes (`SiteConfig`). No behavior change.

### 3.9 `src/server/index.ts` and `src/index.ts`

- Server entry: export `SiteConfig`, `resolveSiteConfig`, `normalizeSiteBaseUrl`,
  `SiteConfigContext`, `SiteConfigProvider`, `setSiteConfigProvider`,
  `createSiteConfigContext`, `createSiteConfigContextFromRequest`. Remove
  `resolveDraftConfig` and `DraftConfig`.
- Isomorphic entry: nothing new (`DraftData` already exported).

### 3.10 `src/server/csp.ts`

No change. Leave the README's CSP table as is.

### 3.11 Tests (Vitest, `packages/headless`)

`config.test.ts` (`describe('resolveSiteConfig')`, rename existing cases):

- normalizes a trailing slash and an install sub-path; keeps port and IP literals
- rejects non-http(s) scheme, credentials, query, fragment, relative value (one case each, message names the part)
- provider-overridden `baseUrl` discards env `CANVAS_JSONAPI_PREFIX`, `CANVAS_JSONAPI_URL`, `CANVAS_JSONAPI_SITE_URL`
- overridden `baseUrl` equal to env (modulo normalization) keeps them
- invalid env `CANVAS_SITE_URL` does not throw when the override supplies a valid `baseUrl`
- `jsonApiProxyPath` from env survives a `baseUrl` override

New `site-config-provider.test.ts`:

- `setSiteConfigProvider` / `getSiteConfigProvider` round trip and `null` clears
- `createSiteConfigContext`: `url` from `getRequestUrl`, `null` without it; `getHeader` returns `null` when the adapter lacks it
- `createSiteConfigContextFromRequest`: header and cookie reads

`flows.test.ts` (extend `makeAdapter()` with `getHeader` backed by a `Map`;
`makeServer` gains an optional `config` of either shape; add a
`beforeEach(() => setSiteConfigProvider(null))`):

- `getConfig`: explicit static overrides; explicit provider receives a context (`url`, header, cookie) and may be async; global provider used when `options.config` is absent; explicit `options.config` suppresses the global provider; `undefined` from a provider falls back to env
- `enableDraftMode`: stored cookie carries `site` equal to the normalized resolved `baseUrl`; token exchange is posted to the provider-selected site
- `getDraftData`: returns `null` for a cookie whose `site` differs from the resolved site, and for a cookie without `site`; returns the session when equal
- `renewDraftSession`: mismatched site → 400 as "no session"
- `getClient`/`getJsonApiRuntimeConfig`: two sites selected alternately by a provider each discover their own prefix (site-data fetched once per site, not once per server) and the runtime config `baseUrl` follows the selection
- `handleJsonApiProxy`: upstream fetch goes to the provider-selected site

`site-data.test.ts`: unchanged behavior; rename type only.

`draft-data.test.ts`: fixture gains `site`; `parseDraftData` rejects a
payload without `site`; round-trip keeps it.

`components-endpoint/handler.test.ts`: default config consults the global
provider with a request-derived context; explicit `config` receives the
context.

## 4. Adapters

Common to all: rename imports/types; implement `getHeader`; rename exported
`getDraftConfig` → `getSiteConfig` (async); README updates (section 6).
Dev-time eager `resolveDraftConfig()` calls become `resolveSiteConfig()`
(they validate env only; fine to keep).

### 4.1 `packages/headless-next`

- `src/adapter.ts:17-36`: add `getHeader: async (name) => (await headers()).get(name)`.
- `src/server.ts`: `export const getSiteConfig = server.getConfig` (async now). Module-level server stays; it consults the global provider automatically.
- `src/route-handlers.ts:41-44`: `createDraftRouteHandlers(config?: Partial<SiteConfig> | SiteConfigProvider)`; doc: "when omitted, the registered provider or `CANVAS_SITE_URL`".
- `src/with-canvas.ts:81-83`: `resolveSiteConfig()`.
- `src/index.ts`: export `getSiteConfig`, `type SiteConfig`; drop the old names. Consider re-exporting `setSiteConfigProvider` and `type SiteConfigProvider` from the adapter index for convenience (do it for all adapters consistently, or none; recommended: re-export from each adapter index so apps import one package).
- `src/middleware.ts`: no change (CSP only).
- Tests: `middleware.test.ts`, `with-canvas.test.ts` adjust names; add a `route-handlers`-level test only if cheap.
- README: see section 6.

### 4.2 `packages/headless-astro`

- `src/adapter.ts:49-53`: extend `AstroDraftContext` with `request?: Request` (both the `Astro` global and `APIContext` have `request`). `createAstroDraftAdapter`: `getHeader: async (name) => context.request?.headers.get(name) ?? null`.
- `src/server.ts:69-71`: `getSiteConfig(context: AstroDraftContext): Promise<SiteConfig>` → `getDraftServer(context).getConfig()`. It now takes the context like the other helpers.
- `src/integration.ts:150-158`: `resolveSiteConfig()`.
- `src/routes/components.ts:29`: no change needed (default consults the global provider).
- `src/index.ts`: export `getSiteConfig`, `type SiteConfig`, re-export `setSiteConfigProvider`/`SiteConfigProvider`; drop old names.
- Tests: `middleware.test.ts` names.

### 4.3 `packages/headless-nuxt`

- `src/runtime/server/adapter.ts:41-66`: `getHeader: async (name) => getRequestHeader(event, name) ?? null` (import from `h3`).
- `src/runtime/server/session.ts:62-64`: `getSiteConfig(event: H3Event): Promise<SiteConfig>` → `getDraftServer(event).getConfig()`.
- `src/module.ts:101-102`: `resolveSiteConfig()`.
- `src/runtime/server/session.ts` exports (the `./server` subpath): add `getSiteConfig`, re-export `setSiteConfigProvider`/`SiteConfigProvider`; drop old names.
- README: document registration via a Nitro plugin (`server/plugins/canvas-sites.ts`, `export default defineNitroPlugin(() => setSiteConfigProvider(...))`).

### 4.4 `packages/headless-tanstack-start`

- `src/adapter.ts:41-64`: `getHeader: async (name) => getRequest().headers.get(name)`.
- `src/server.ts:28`: `export const getSiteConfig = server.getConfig`; line 68 `export { resolveSiteConfig }`.
- `src/route-handlers.ts`: optionally accept `config?: Partial<SiteConfig> | SiteConfigProvider` to match Next; doc line 51 reworded.
- `src/vite.ts:17`: `ENV_KEYS` only bridges two keys; leave as is (out of scope) but do not regress.
- `src/vite.ts:102-104`: `resolveSiteConfig()`.
- `src/index.ts`: export `getSiteConfig`, `type SiteConfig`, re-export provider API; drop old names.
- Tests: `route-handlers.test.ts`, `middleware.test.ts` names.

### 4.5 `packages/headless-angular`

- `server/index.ts:59-75` inline adapter: `getHeader: async (name) => request.headers.get(name)`.
- `CanvasServerOptions` (29-35) type picks up the new `config` type automatically.
- Metadata handler (121-125): `config: () => server.getConfig()` still type-checks if the handler accepts a Promise-returning function (3.7). Keep.
- No `getDraftConfig` export exists; nothing to rename publicly. `tests/build-output.test.ts` unaffected.

### 4.6 `packages/headless-react`

No references; no change.

## 5. Behavior notes for the implementer

- Resolution happens on every call; nothing is captured at construction except the per-site discovery resolvers.
- The cookie-site comparison and the discovery map both key on the already-normalized `SiteConfig.baseUrl`, so there is exactly one normalizer.
- Keep credentials out of `SiteConfigContext` and out of anything logged.
- The selection rule must be computable from every request type (page, proxy, draft routes, components endpoint). Document that a rule based on the page path alone cannot work because proxy requests carry no page path; host, cookie or header rules do.
- `getDraftData()` resolving config only when a cookie is present must be preserved: Astro/Nuxt/Angular call it from CSP/finalize paths.

## 6. Documentation

- `packages/headless/README.md`
  - 194-203, the paragraph "Configuration: `CANVAS_SITE_URL` is the backend…": rewrite to say `CANVAS_SITE_URL` is the default site; a `SiteConfigProvider` registered with `setSiteConfigProvider()` (or passed as `config`) selects a site per request from the request URL, headers or cookies; the provider's `baseUrl` replaces the env JSON:API overrides; the proxy, token exchange, renewal and runtime config follow the selected site; draft sessions are bound to their issuing site and read as "no session" elsewhere; selection must be derivable from every request (host/cookie/header, not page path); never derive `baseUrl` from untrusted input because the selected site is the token verifier. Include the neutral provider example from section 2.
  - 274-…, "Writing a framework adapter": step 1 mentions the optional `getHeader`; step 6 mentions `getConfig()` is async.
  - Keep the CSP table untouched.
- `docs/user/src/content/docs/headless/setup.mdx`: lines 52, 129 ("`CANVAS_SITE_URL` is the entire …") and 264 ("`CANVAS_SITE_URL` is the whole configuration…"): soften to "the default site" with one sentence pointing to per-request selection in the package README. Keep the narrative otherwise.
- `docs/user/src/content/docs/headless/multilingual-sites.mdx`
  - Table row "URL domain" (line 35): "Not supported" with no issue link.
  - Bullet at line 42: replace with: Drupal-side domain-based language negotiation is not supported. The SDK can select a Drupal site per request, but Canvas editor previews and assertion handling do not support per-language domains. Do not mention 3592072.
- Adapter READMEs: Next 15/192-197, Astro 19/110-115, Nuxt 15/93-98, TanStack 15/144-149, Angular 78-79/145-147: `CANVAS_SITE_URL` is the default site; one-line registration snippet for that framework; "fetched once per server instance" → "fetched once per site".
- `.cspell.json`: add any new words flagged by `npm run lint:cspell`.

## 7. Changesets

`.changeset/README.md` and `scripts/changeset-prompt.js` say never to use
`major` and to flag breaking changes. This is pre-decided: all packages are
0.x, and the user chose a breaking rename in a `minor`. Do not stop to flag it.
Write one changeset per package (texts differ). Omit issue numbers.

`@drupal-canvas/headless` (minor), first line imperative, then bullets:

```
Select the Drupal site per request with a `SiteConfigProvider`, keeping `CANVAS_SITE_URL` as the default.

- Breaking: `DraftConfig` is now `SiteConfig` and `resolveDraftConfig()` is now `resolveSiteConfig()`; `DraftServer.getConfig()` returns a promise.
- `setSiteConfigProvider()` registers a provider every adapter-created server consults; `createDraftServer({ config })` also accepts one. Providers receive the request URL, headers and cookies and return a partial `SiteConfig` or `undefined` for the default site.
- A provider-selected `baseUrl` replaces the environment's JSON:API overrides for that request.
- `resolveSiteConfig()` validates and normalizes the site base URL.
- Draft sessions record the site that issued them and are not presented to other sites; sessions created before this version must be reactivated.
- Framework adapters may implement `getHeader()`.
```

Adapters (minor), e.g. Next:

```
Select the Drupal site per request and rename `getDraftConfig()` to `getSiteConfig()`.

- Breaking: `getDraftConfig` is now `getSiteConfig` and returns a promise; the `DraftConfig` type is now `SiteConfig`.
- Register a `SiteConfigProvider` with `setSiteConfigProvider()` in `instrumentation.ts`; `createDraftRouteHandlers()` accepts a provider too.
```

Astro and Nuxt additionally: `getSiteConfig()` now takes the request
context/event. Angular: no public rename; mention `getHeader` support and the
provider.

## 8. Manual verification: two local Drupal sites, one frontend

Record the steps and observed results in a "Manual verification" section of
the MR description. Prerequisites in this environment: `composer install`,
`site-install`, `canvas-env-start` (see `AGENTS.md`). The default site is at
`http://127.0.0.1:8080` (also `http://localhost:8080`).

### 8.1 Second site

Set up a second, independent Drupal site next to the default one, reachable
at its own URL (for example `http://localhost:8081`). How is up to the
implementing agent; Drupal multisite under the existing docroot with a
second database and a second PHP built-in server is the obvious route, and
the setup is runtime state, not a change to the provisioned scripts. The
second site must:

- be installed the same way `site-install` installs the default site: same
  modules, the `canvas_dev_oauth` and `canvas_dev_humanify` recipes, OAuth
  keys and the `cli` consumer;
- serve JSON:API under a different prefix (`/api` instead of `/jsonapi`), so
  the per-site discovery cache is visibly exercised;
- have visibly different content (site name and one article title are
  enough);
- have the frontend `http://localhost:3000` registered under Headless
  frontends, with the component sync run from its editor. The default site
  needs the same registration and sync.

### 8.2 Frontend (Next template with local packages)

1. In the Canvas checkout: `npm install`, `npm run packages:build`, then `npm pack` in `packages/headless`, `packages/headless-next`, `packages/headless-react` (tarballs avoid duplicate React copies that symlinks cause).
2. In a copy of `~/workspace/headless-templates/nextjs`: install the three tarballs, `.env` with `CANVAS_SITE_URL=http://localhost:8080`.
3. Add `canvas-sites.ts` with a cookie-selected provider so both sites share one frontend host and the session-binding path can fire (a host-based rule never can, because cookies are partitioned per host):
   ```ts
   export const siteConfigProvider: SiteConfigProvider = async ({ getCookie }) =>
     (await getCookie('canvas-site')) === 'second' ? { baseUrl: 'http://localhost:8081' } : undefined;
   ```
   Register it in `instrumentation.ts` next to `trustSystemCertificates()`.
4. `npm run dev` on port 3000. On `localhost`, ports do not separate sites, so cookies set on `localhost:3000` are also sent inside the editor iframe at `localhost:8080`/`8081`; set the test cookie in DevTools with `SameSite=Lax` or `None`.

### 8.3 Scenarios and expected results

| #   | Steps                                                                                                             | Expected                                                                                                                                                    |
| --- | ----------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1   | No cookie, open `/`                                                                                               | Site 1 published content; network shows `localhost:8080`.                                                                                                   |
| 2   | Set `canvas-site=second`, reload                                                                                  | Site 2 content; JSON:API proxy requests upstream to `localhost:8081/api/...` (prefix discovered per site; logs show two site-data fetches total).            |
| 3   | Clear cookie; open site 1's Canvas editor, preview a page                                                         | Draft markup and editor overlays present; `canvas-draft-data` cookie JSON contains `"site":"http://localhost:8080"`.                                         |
| 4   | With that session, set `canvas-site=second`, reload the page in the top-level frontend tab and in the editor iframe | Site 2 published content; no Canvas boundaries in DOM; no `Authorization` header on any request to `:8081`; draft cookie still present but ignored.           |
| 5   | Remove the cookie, reload                                                                                         | Site 1 draft preview resumes without reactivation.                                                                                                          |
| 6   | Keep `canvas-site=second`; open site 2's editor, preview a page                                                   | Activation against `:8081/oauth/token`; cookie now `"site":"http://localhost:8081"`; site 2 drafts render.                                                  |
| 7   | Remove the cookie while the site 2 session exists; reload                                                         | Site 1 published content, no token sent to `:8080`.                                                                                                         |
| 8   | Start the app with an old-format draft cookie (edit the JSON to remove `site`)                                    | Treated as no session; reopening the editor reactivates.                                                                                                    |
| 9   | Provider returns `{ baseUrl: 'ftp://x' }` or a URL with credentials                                               | Request fails with the descriptive configuration error (500 `configuration_error` on proxy/components routes; thrown error on page render).                 |

Capture the DevTools network evidence for 4 and 7 (screenshots are enough).

## 9. Commands and checks

From `$CANVAS_ROOT`:

- `npm run packages:type-check`
- `npm run packages:test` (or `npx vitest run` in the package)
- `npm run lint` (cspell, eslint, prettier, stylelint, yaml, type-check)
- `npm run fix` for prettier/eslint autofix
- Changesets: `npx changeset` or hand-write files in `.changeset/`.

Do not use Canvas's `composer run …` scripts (see `AGENTS.md`). PHP is not
touched by this work.

## 10. Out of scope (do not do)

- Any PHP change in `modules/canvas_headless` or `src/`.
- Drupal-side domain language negotiation (four known blockers: absolute `path` claim in `AssertionController::resolvePath`, possibly absolute `aud` in `PreviewAssertionFactory::tokenEndpointAudience`, cross-domain 404 in `PreviewLanguageSubscriber`, no frontend-to-language pairing in the `frontends` config).
- Renaming the server tier (`createDraftServer`, `DraftServer*`).
- Deprecated aliases for the renamed config API.
- A declarative host-to-site map, an allowlist of sites, CSP changes.
- Custom request headers through `fetchPage` (work item 3592073).
- Updating the Angular template in `headless-templates` (the user does this).
- Making the second-site setup repeatable in `canvas-hazelnut` (`site-install`); optional separate change.
