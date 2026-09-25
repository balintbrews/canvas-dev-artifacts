# Canvas MR !1695 — scoped QA

Source: f442e064defe6e5a0b6ea5bbc3c83a7e0fc59b32. Drupal 11.3.10, Next.js 16.2.12, MR-built SDK packages. Sessions: 24–25 September 2026.

## Confirmed defects

- F1: Existing unsaved node preview omits core preview behaviors. Teaser does not submit; embedded navigation leaves without confirmation. Cancel/Leave cannot be reported as passing.
- F2: A canonical preview sets the shared frontend cookie to exclude autosaves. An editor in another tab then renders saved content despite its newer autosaved form value. Refresh interference confirmed; timed renewal and reverse direction unverified. Private-hostname screenshot withheld.
- F3: New unsaved node preview works monolingually, then fails after adding French. Translation links attempt to create a URI for an entity without an ID: API 500, visible frontend Not found. No translation was created.

## Passing scoped checks

Canonical page/node, older revision, latest moderated draft excluding unrelated autosave; repeated unsaved existing-node previews and back-to-edit restoration. Authenticated component sync: 18 components, no warnings/errors. Anonymous and no-preview account fallback have no iframe. Read-only account previews but cannot edit. Real preview token GET succeeds; PATCH denied 403 while JSON:API writes temporarily enabled; configuration restored and stored title unchanged. Missing metadata assertion 401; mismatched origin 403; valid sync 200.

Drupal and frontend-only link navigation works. External Drupal.org navigation reaches its challenge, not validated content.

## Tests and limits

Next lint/typecheck/build passed. Package suites: SDK 287, Next 19, React 11, Astro 8, Nuxt 10, TanStack 7, Angular 1. Kernel: 5 tests/310 assertions and 48 tests/4096 assertions, with 28/32 deprecations. Only Next has application runtime coverage. Package tests do not establish runtime correctness of the other four adapters.

Local transport-only workarounds address actual proxy duplicate ACAO and warning headers; they retain authentication/origin validation and are not MR fixes. The preexisting Next README proxy shorthand build problem is excluded from MR findings. No workaround, private preview hostname, credential, signed URL or traffic dump is published here.

## Videos

Native 1920×1080, 30fps, square pixels, intact 1920×996 browser page beneath an 84px external branded bar. Silent; real-time interaction; ends-only trims; one-second final hold. Render plans and actual monotonic milestone logs accompany each video. Caption times were checked against visible frames, not merely command timestamps. Preliminary duplicate-cursor and incorrect-selection takes were rejected, not delivered.

This local collection is evidence, not merge approval. QA did not apply implementation fixes or publish this artifact collection. MR comments follow a separate human-approved workflow.
