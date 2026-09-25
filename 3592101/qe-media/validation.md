# Incremental QE evidence validation

## Case 1 — complete evidence pair

Source: `ecf023f6c1af88b2014912ee03b2c559a0303d13`. Local checkout clean and remote issue branch unchanged at this milestone. No Canvas fixes or pushes.

| Clip | Duration | Format | SHA-256 |
| --- | --- | --- | --- |
| `case-1-terminal.mp4` | 21.8 s | H.264, 1920×1080, 30 fps, SAR 1:1, DAR 16:9 | `26dc327704b2dac3b3d503ca87b971fe295c15b679fcc54bf9db65cef7ff4f5a` |
| `case-1-browser.mp4` | 57.3 s | H.264, 1920×1080, 30 fps, SAR 1:1, DAR 16:9 | `3a927e6a2a9e25459505e76a7878681ac11626069862e22b832a66c354ca1a31` |

- Successful final-source rehearsals preceded both captures.
- Terminal clip honestly verifies an existing installation. Initial reconciliation and push are recorded in the scenario checks, not replayed in this video.
- Each full MP4 decoded without errors and played through to `ended` in Chromium at normal speed, without a media error.
- Native 1920×996 content retained beneath the 84px title bar. SVG mark, title and caption regions visually checked. No cropping, internal cuts or speedups. Renderer adds a one-second final hold.
- Browser interaction review covered seven approach/press/release sequences, including exact adjacent 30fps frames and travel samples. Card slot exit/return, root Spacer reorder, and three code previews are visible. No pointer jumps observed.
- Full milestone frames reviewed for text readability, caption intent, visual state, full content frame and secret exposure. Terminal output includes the expected ignored-legacy-regions warning; it is not hidden or called a region migration pass.
- Milestone logs and render plans accompany both clips. Captions are action cues, so they begin as the pointer approaches the next action. Monotonic logs document the observed command/readiness times; plans were checked against visible frames.
- Report and both media files also loaded successfully through the actual supplied HTTPS preview route. Expiring routes are intentionally not embedded in artifacts. Full playback testing was on the local report origin.
- Desktop 1920px and mobile 390px layouts checked; mobile had no horizontal overflow. Navigation targets resolve. Automated WCAG 2 A/AA checks reported zero violations. An incomplete automated check concerned the video fallback download link; links remain explicitly underlined. This is not a claim of comprehensive accessibility certification.
- Original successful raw video/timeline inputs are removed only after this validation. Private review stills and failure/debug logs remain outside the public artifacts checkout.

## Case 2.1 — actual 1.11.0, complete evidence pair

Server: `ef76aaf171331471a78fa12d151ead50a5270ec6` (actual 1.11.0). Clients: preserved `ecf023f6` MR artifacts. Published legacy fixture: `c700e6ff91ec5570ef966be541083da465de0661`.

| Clip | Duration | SHA-256 |
| --- | --- | --- |
| `case-21-terminal.mp4` | 21.833 s | `0fdd877762fc1aa8c3749f2c20f21be03a8024a38207e79ceef4b6b6d395cfbd` |
| `case-21-browser.mp4` | 28.567 s | `c6f06d8d6c307156d2a28fc1e8397975f4bd7d511532e4e21d3e15ce52ad870d` |

- User explicitly approved `--no-include-brand-kit` for Case 2.1 only; the guide now explains the Folder OAuth incompatibility and retained component/page coverage. No authentication or server source workaround.
- Retained clean released seed verified; actual pull, mandatory reinstall/artifact/source verification and push repeated successfully in terminal footage. Original install/seed push are not reenacted.
- Browser footage verifies actual 1.11.0 Workbench fallback branding/theme/page context, Home menu, decoded image, all three articles and formatted bodies, then the persisted Drupal round-trip with real site/page metadata. No alerts.
- Both full files decode without error and play to ended in Chromium at normal speed without media errors. H.264, 1920×1080, 30fps, SAR1:1; complete native 1920×996 content under the84px header.
- All native milestone/end frames, seven browser interaction sheets and supplementary adjacent before-frame strips visually reviewed. Smooth scrolling and natural page loading retained; no cuts, speedups or pointer jumps. SVG, title/captions and final one-second hold checked.
- Recorded intentional old-server capability/AI advisory, static-config fallback401 messages, legacy-regions and npm install-script advisories remain visible. These are not per-component getter-conversion/manual-client warnings. Source files remain byte-identical.
- New successful raw inputs were deleted after completed validation. Plans, milestone logs, checks and private review stills remain. No failure evidence is presented as success.
- Updated report has four players and zero mobile horizontal overflow. A long historical SHA wrapping issue was fixed. Local WCAG2 A/AA audit: zero violations, one incomplete fallback-link check; no comprehensive certification claimed.
- Local report/media loading passed. The previously supplied HTTPS route now returns proxy401: Invalid or expired token. A fresh port3000 preview route is needed; this is not a Canvas or media failure.

## Case 3.1 — accepted original evidence restored

Exact delivered frames at 14.433–14.467 seconds show both successful getter conversions and constructor-only per-file warnings, matching the guide screenshot. User explicitly accepted the original video after this audit; no retake is required. Both original players and source identities are restored. The generic all-API summary wording was tracked separately at that checkpoint; its narrow fix was subsequently reviewed, published and preserved through the authorized rebases.

Server and clients: original `ecf023f6c1af88b2014912ee03b2c559a0303d13`. Both captures completed before the approved typing-only fix was transferred into main Canvas.

| Clip | Duration | SHA-256 |
| --- | --- | --- |
| `case-31-terminal.mp4` | 26.900 s | `0f4fb36a7461cb115d0b2acdc6316b0455429a84930abaeb4d11e889d69f8bd0` |
| `case-31-browser.mp4` | 28.567 s | `c5008bc0ef07f691819af9377f44d05416b7263548f53d1393ba6e7b0ccfdcf1` |

- Successful rehearsal before capture. Terminal truthfully repeats restoration of the retained released legacy seed, default pull, reinstall/package verification and push. Exact getter/import transformations checked; both constructors preserved with the expected migration warnings. No Brand Kit opt-out in this case.
- Browser verifies live Workbench name/slogan/logo, Home, decoded image and all three formatted article bodies, then Drupal real page title/UUID and the full tree. Workbench page/entity context remains absent as expected. No alerts.
- Full decode and normal-speed Chromium playback to ended passed for both files, without media errors. H.264,1920×1080,30fps,SAR1:1. Native1920×996 content retained beneath84px header, packaged SVG mark, measured captions, no internal cuts/speedups and one-second final hold.
- All native milestone/end frames, seven interaction sheets and adjacent before-frame strips reviewed. Smooth scrolling and natural navigation/loading retained. Owned successful raw inputs removed only after validation; plans, milestones and private review stills retained.
- Fresh supplied HTTPS report route works. Six players; automated WCAG2 A/AA audit reports zero violations and one incomplete fallback-link check. New long integration SHAs needed wrapping in the mobile preparation lead; fixed without changing runtime source. No comprehensive accessibility certification claimed.

## Case 3.3 — portable default frontend, complete evidence pair

Recorded server/client source: `68ef5df9335769777246ec492b8dc03314ab9d5d`; portable fixture `6288ed2261d5861f2827f4102e7be91e00d0bb66`. These videos predate the rebase and are not relabeled.

- Terminal: 19.467s; SHA256 `78237941a7b2044496d4585089f2fcc8d7ce7dcb173ff7275316bb3654dc2c73`.
- Browser: 28.000s; SHA256 `8451db983fad6ec03dea3533fec2f2e4133c4b1311d6eb99eb37d0fdd84d14ca`.
- Decode and native1920×1080/30fps/SAR1:1 checks passed; both normal-speed playback records reached ended without error. Native milestone/end frames and seven interaction sheets inspected. One-second final hold retained.
- Clean setup/seed push preceded capture. Terminal discloses repeat pull/reinstall/verification/push against retained portable seed. No getter conversion or manual-client warnings; unrelated regions notice remains. Both sources unchanged and installed committed packages byte-verified.
- Browser verifies metadata, menu, image, all three dated rich-text articles and correct Workbench/Drupal page-context differences, with zero alerts. Raw evidence remains retained.

## Clarification: Case 2.1 static-config 401

The two warnings in `approved-21-push.log` came from the CLI's bundled Vite plugin (`packages/vite-plugin/src/index.ts`, buildStart). It anonymously requests `/canvas/api/v0/site-data` with credentials omitted; actual1.11.0 requires authentication there. Static fallback supplies configured base URL and optional JSON:API prefix, without fetched branding/theme assets. This is not the authenticated component-upload response: both component updates, CSS and page update succeeded and push exited0. Drupal round-trip verified real metadata. The same metadata path explains Workbench's old-server branding fallback. Case3.1's MR-server push has no such401 warnings. No destructive rerun was used for this clarification.

## Case 3.2 Next — development-runtime evidence complete

Recorded Canvas/server packages: `cbf2d0967e012cd705e27d73343faeedacba50ea`; template `43fb805e3442ef6ab2381f4dc107c153806c43f6`; legacy fixture `c700e6ff91ec5570ef966be541083da465de0661`.

- Terminal: 51.833s; SHA256 `c5a60a403ce8341aba72370177d58e3e0d727f89c47df9eff72be2d47684e93e`.
- Browser: 34.033s; SHA256 `ee70200eca4f3ea1b50e2b1e1824ce708c4c79e0fbc81f2c0f56d43ae6a8e1b2`.
- Both full files decoded and played to ended at normal speed in Chromium. Native 1920x1080/30fps/SAR1:1 output retains the 1920x996 content and 84px header, packaged mark and one-second final hold.
- All milestone states and seven adjacent interaction sheets reviewed, including native getter-conversion/warning, client-diff and final frames. Actual seed restoration/pull, package reinstall/verification, both remaining constructor migrations and lint/type checks are visible. No fabricated terminal output, internal cuts or speedups.
- Public and embedded Canvas draft trees visibly render site/page metadata, Home menu, the 16x16 checker fixture, three dated articles and rich-text bodies. Natural loading and scrolling retained. Local component-sync success is visible. No signed URLs in the captures.
- Production qualification: original template43fb805 fails production build on its re-export-only proxy. Later reviewed template92470b79 passes checks/build and separate local production public/draft request smoke with coherent64d packages; metadata sync succeeds and anonymous metadata stays401. Neither capture claims that later source or production runtime.
- Temporary edge CORS and development-origin changes were removed before these local captures and source publication. Exact-origin proxy diagnostics remain private. Raw evidence and the prior report snapshot are preserved.

## Case 3.2 TanStack — development-runtime evidence complete

Source64d932aec1d731389a81e1cfa0eee50747fab555; template92470b79d1e86b8c5ac4c7852a068a58641ee573. Fresh published legacy archive and newly reconciled checker image were used after rejecting an initial stale-media seed attempt.

- Terminal54.800s: SHA256 `9c6514b0103bb2d8f41ba00e91afea8b265f4abf7bef773ea76afc4dec58c9c6`.
- Browser34.267s: SHA256 `4c96ab779022c33a1df1645179183444e100ce9887e65511b7341e0ed395b844`.
- Both full files decode and play to ended at normal speed; native1080p/30fps/SAR1:1, original996px content and84px header, one-second final hold. Milestone/diff/end frames and all seven adjacent interaction sheets visually reviewed. Readable conversion/warning and manual-client diff holds retained; unrelated npm install-script warnings remain visible.
- Actual public and embedded draft trees show metadata, menu,16x16 checker image and three dated rich-text articles. No source/CORS workaround, signed URLs, hidden cuts or speedups. Initial install/reconciliation precede the honest repeat-pull terminal recording.
- Prior report snapshot and unsuccessful stale-seed evidence preserved privately. Report now has12 players; automated WCAG2A/AA audit reports zero violations and one incomplete fallback-link check, not comprehensive certification.

## Case 3.4 — portable Next and TanStack, complete evidence pairs

Both recorded at Canvas `64d932aec1d731389a81e1cfa0eee50747fab555`, template `92470b79d1e86b8c5ac4c7852a068a58641ee573`, portable fixture `6288ed2261d5861f2827f4102e7be91e00d0bb66`.

| Clip | Duration | SHA-256 |
| --- | --- | --- |
| `case-34-next-terminal.mp4` | 36.733s | `dd0603cf84ee5c5e3b56211dfa9bcc97466c2bdc1e0004b5527df680dc615801` |
| `case-34-next-browser.mp4` | 35.567s | `db85f3e25c335d33b4f1b2c8847aeba3402c18bc220c7b4b4f4ea22e3b6a6044` |
| `case-34-tanstack-terminal.mp4` | 38.833s | `ab2a46a1c5b6e4ae89a363c50fe2c8f781a26a4e582be58436217d0965e43a79` |
| `case-34-tanstack-browser.mp4` | 34.733s | `ad38f7a569969b0c2c35677c213a885cf5f441467c3ca841b2784a46d00a9ed0` |

- Each case used a clean Drupal installation and fresh archive reconciliation before capture. Terminal recordings truthfully repeat seed push/pull, mandatory reinstall, full committed-package byte verification and lint/TypeScript checks. Both component sources remain byte-identical to the published portable fixture. No getter conversions or manual-client warnings; unrelated regions and install-script notices remain visible.
- Actual public frontends and embedded Canvas draft previews show live metadata, page title/UUID, Home menu, decoded16x16 checker image and all three dated rich-text article bodies. Both frameworks separately build successfully. Captured development runtime is not a broad production certification. Anonymous TanStack metadata remains401.
- All four files fully decode and play to ended at normal speed; 1920x1080/30fps/SAR1:1, native1920x996 content plus84px header, packaged mark and one-second final hold. Native milestones/end frames and seven interaction sheets per browser clip reviewed. No hidden cuts, speedups, origin workarounds or signed URLs.
- Next terminal's final caption specifically says no getter/client warnings rather than suggesting an entirely warning-free CLI. Caption-only draft retained privately; final file was re-decoded, visually checked and replayed to ended. Underlying terminal footage was not changed.
- Prior6/8 and7/8 reports preserved. Original accepted3.1 footage and all earlier identities remain unchanged.

## Coverage boundary

All eight executable guide cases are evidence-complete at their recorded revisions (8/8; sixteen players), not broad production or release certification. Case3.1's accepted original footage is unchanged. Section2.2 is unsupported/N/A. Historical clips are not relabeled as latest-source coverage. Both legacy and portable Next/TanStack scenarios include actual public and embedded draft runtime checks, not merely package builds or decoding.

The separately reviewed local development environment change retains both PHP upload limits at 32M. It remains uncommitted and is not part of the Canvas MR.

## Final report audit

- All16 delivered MP4 hashes match the scenario manifest; completed normal-speed playback and milestone/content reviews remain retained. No scenarios or videos were repeated on resume.
- Report restarted on0.0.0.0:3000 and locally returned200. All69 linked report resources returned200; guide navigation targets resolve. All16 embedded players load1920x1080 metadata and non-empty caption tracks without media errors.
- Automated WCAG2 A/AA scan: zero violations, one incomplete video-fallback-link check. Keyboard skip-link and native player play/pause checks passed. This is not comprehensive accessibility certification.
- Desktop1920px and mobile390px layouts reviewed. An unbroken historical SHA in the preparation list caused mobile overflow; inherited text wrapping fixed it, and the final audit confirms no horizontal overflow.
- Original Case3.1 videos and all per-scenario identities are unchanged. Failed takes remain outside the presentation. Temporary Next runtime-origin/CORS hunks are absent; Canvas and template worktrees remain clean.
- The verification browser was closed. Report-only serving is retained for review; fresh external preview access requires Hazelnut coordination. No artifact Git publication or credential borrowing occurred.
