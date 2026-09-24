# #3592101 Testing guide

_Written without the use of AI_.

MR: https://git.drupalcode.org/project/canvas/-/merge_requests/1666

## Prerequisites

1. Clone the following two repositories:
   1. https://github.com/balintbrews/canvas-3592101-legacy
   2. https://github.com/balintbrews/canvas-3592101-portable
2. Run `npm install` in both directories.
3. Copy their `.env.example` files in both as `.env`, and connect to your Drupal
   site that way.

## Terminology

### Canvas 1.11.0

A Canvas version that DOES NOT have the changes from this branch. The latest
tagged version before that is `1.11.0`, but you can use any version from `1.x`
prior to merging this branch.

### Canvas with #3592101

A Canvas version with changes from this branch. If you're testing before merging
this branch, that means a checkout of this branch, if you're testing afterwards,
that means a Canvas version — either a tagged version or a ref from `1.x` — that
already has these changes integrated.

### Local codebase

The codebase where your Code Components live, not the Canvas codebase. A local
codebase can be a default frontend or a headless frontend codebase.

### JS packages from #3592101

Refers to packages installed in your Local codebase (see above).

The JavaScript packages are released independently of Canvas itself, so we'll
need to test combinations. When the guide refers to JS packages from #3592101,
it means we're looking to use the `drupal-canvas` and `@drupal-canvas/*` npm
packages that contain the changes from this branch when testing with a local
codebase. This means the

Prior to merging and releasing the JS packages from this branch, this means you
need to check out the branch, do an `npm run build`, pack them with `npm pack`,
then update your `package.json` file in your local codebase to point at the
packed versions. Give this prompt to your coding agent, and it will do this for
you: https://gist.github.com/balintbrews/6b2355b6d74cc07cc721b618a1aa4dec.

After merging and releasing the JS packages from this branch, be sure to use the
NEW package versions. Look for a commit in `1.x` with the subject of "chore:
version packages" which was created after the merge commit. That commit will
show you the NEW JS package versions as tags.

### Older JS packages

Refers to packages installed in your Local codebase (see above).

When the guide refers to Older JS packages, make sure to use released versions
of the `drupal-canvas` and `@drupal-canvas/*` npm packages prior to this branch.

Prior to merging and releasing the JS packages from this branch, this means the
latest tagged versions that an `npm install` gives you.

After merging and releasing the JS packages from this branch, be sure to use the
OLD package versions. Look for a commit in `1.x` with the subject of "chore:
version packages" which was created after the merge commit. That commit will
show you the NEW JS package versions as tags, so you'll need to use prior
versions to those.

### Clean Drupal installation

A clean Drupal installation means that the Drupal instance DOES NOT have any
Code Components. Note that a Drupal instance may have Code Components even if
none is visible under the _Code_ tab in Canvas: A connected headless frontend
can leave components as `type: external` which will not be listed once the
headless frontend is disconnected. The safest way is to re-install your Drupal
whenever this guide asks for a clean Drupal installation.

Whenever you re-install Drupal, please be sure to run `npm run build` from the
Canvas module's directory.

Set a site name and slogan at `/admin/config/system/site-information`.

Generate example nodes for your installation. Use the Devel Generate module:

```bash
   ddev composer require drupal/devel && \
     ddev drush pm-install devel -y && \
     ddev drush devel-generate:content
```

### Example component tree

The test repositories provide an example page with example components that
output the followings:

- Site name
- Site slogan
- Theme logo URL
- Site menu (You may only see "Home" as the only menu item.)
- Current page title
  - Expected to be missing in Workbench
- Current entity UUID
  - Expected to be missing in Workbench
- A small example image outputted with the `Image` component
- A list of example articles (see Clean Drupal installation above for generating
  them)
  - Their body text is rendered by `FormattedText`

### Code update/warning

When code from the `legacy` repository gets automatically updated during an
`npx canvas pull`, here is what to expect:

1. `article-list` comonent: `getPageData()` replaced by `usePageContext()`
   imported from `drupal-canvas/react`.
2. `page-frame` component: `getSiteData()` replaced by `useSiteContext()` from
   `drupal-canvas/react`.
3. `new JsonApiClient()` calls remain unchanged and receive manual-migration
   warnings.

## Scenarios

### 1. General smoke test

1. Start with a Clean Drupal installation using Canvas with #3592101.
2. Create a local codebase with Nebula (`npx @drupal-canvas/create@latest`).
3.

### 2. Testing with Canvas 1.11.0 and JS packages from #3592101

#### 2.1. Default frontend

1. Run `npx canvas reconcile-media -y && npx canvas push` from the `legacy`
   repository to populate a Clean Drupal installation using Canvas 1.11.0.
2. Verify the Example component tree (see above) in Drupal.
3. Create a local codebase with Nebula (`npx @drupal-canvas/create@latest`).
   1. Adjust it to use JS packages from #3592101.
4. Run `npx canvas pull`.
5. **Code update/warning (see above) MUST NOT be happening.**
6. Verify the Example component tree (see above) in Workbench.
7. Run `npx canvas push`.
   1. The command should execute cleanly.
8. Verify the Example component tree (see above) in Drupal.

#### 2.2. Headless frontend

Pulling the components from the example repos into a headless codebase similar
to _1.1. Default frontend_ is not supported with Canvas 1.11.0. Nothing to test
here.

### 3. Testing with Canvas with #3592101 and JS packages from #3592101

#### 3.1. Default frontend starting from legacy components

1. Run `npx canvas reconcile-media -y && npx canvas push` from the `legacy`
   repository to populate a Clean Drupal installation using Canvas with
   #3592101.
2. Verify the Example component tree (see above) in Drupal.
3. Create a local codebase with Nebula (`npx @drupal-canvas/create@latest`).
   1. Adjust it to use JS packages from #3592101.
4. Run `npx canvas pull`.
5. **Code update/warning (see above) MUST be happening.**
6. Verify the Example component tree (see above) in Workbench.
7. Run `npx canvas push`.
   1. The command should execute cleanly.
8. Verify the Example component tree (see above) in Drupal.

#### 3.2. Headless frontend starting from legacy components

**Repeat** these steps **for all five headless frameworks**: (1) Next.js, (2)
Astro, (3) Nuxt, (4) TanStack Start, and (5) Angular.

Run `npx canvas reconcile-media -y && npx canvas push` from the `legacy`
repository to populate a Clean Drupal installation using Canvas 1.11.0. 2.
Verify the Example component tree (see above) in Drupal. 3. Create a local
headless codebase
(`npx @drupal-canvas/create@latest --experimental-headless`). 1. Adjust it to
use JS packages from #3592101. 4. Run `npx canvas pull`. 1. Answer _yes_ to
deleting the unused components from your local codebase. 5. **Code
update/warning (see above) MUST be happening.** 6. Verify the Example component
tree (see above) in Drupal.

#### 3.1. Default frontend starting portable components

1. Run `npx canvas reconcile-media -y && npx canvas push` from the `portable`
   repository to populate a Clean Drupal installation using Canvas with
   #3592101.
2. Verify the Example component tree (see above) in Drupal.
3. Create a local codebase with Nebula (`npx @drupal-canvas/create@latest`).
   1. Adjust it to use JS packages from #3592101.
4. Run `npx canvas pull`.
5. **Code update/warning (see above) MUST NOT be happening.**
   1. The code should already be in the updated form.
6. Verify the Example component tree (see above) in Workbench.
7. Run `npx canvas push`.
   1. The command should execute cleanly.
8. Verify the Example component tree (see above) in Drupal.

#### 3.2. Headless frontend starting from portable components

**Repeat** these steps **for all five headless frameworks**: (1) Next.js, (2)
Astro, (3) Nuxt, (4) TanStack Start, and (5) Angular.

Run `npx canvas reconcile-media -y && npx canvas push` from the `portable`
repository to populate a Clean Drupal installation using Canvas 1.11.0. 2.
Verify the Example component tree (see above) in Drupal. 3. Create a local
headless codebase
(`npx @drupal-canvas/create@latest --experimental-headless`). 1. Adjust it to
use JS packages from #3592101. 4. Run `npx canvas pull`. 1. Answer _yes_ to
deleting the unused components from your local codebase. 5. **Code
update/warning (see above) MUST NOT be happening.** 1. The code should already
be in the updated form. 6. Verify the Example component tree (see above) in
Drupal.
