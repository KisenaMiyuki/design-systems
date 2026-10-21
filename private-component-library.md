# Packaging a Private React Component Library

Goal: turn the design systems in this repo into React component libraries for private use, both on the same machine and on other machines through a private GitHub repo, while keeping public publishing possible later.

There are two separate jobs:

1. **Porting.** The design systems are static HTML, CSS and JS today (about 14k lines, with canvas and `requestAnimationFrame` animations). They have to become React components with a deliberate API. This is most of the work.
2. **Packaging.** Turn those components into real packages with a clear boundary: a name, a version, a defined public API, and React listed as a peer dependency. Once you have that, "private now, public later" mostly comes down to which registry you publish to.

## Copying JSX files vs. a package

Copying `.jsx` files into projects works, but it has problems:

- Each project ends up with its own drifting copy, and you can't tell which version it has.
- Nothing separates the public components from internal helpers, so projects start importing internals.
- Styles, tokens and dependencies aren't tied to the components, so you have to remember to copy them too.
- Updating means copying again by hand.

A package fixes all four: projects write `import { Button } from '@kisenamiyuki/pixel-flow'`, pin a version, and upgrade on purpose.

## Repo strategy: a dedicated private monorepo

Make a dedicated **private** repo for the libraries and keep `design-systems` as the public showcase.

**The main reason is visibility.** GitHub sets public or private per repository, so you can't have a private folder inside a public repo. `design-systems` has to stay public for the demo pages, because GitHub Pages on a private repo needs a paid plan.

**It also helps for other reasons:**

- **Linked packages:** a package published to GitHub Packages links to its repo and can inherit that repo's access rules.
- **Separate history:** library releases, tags and changelogs stay apart from demo-site changes.
- **Going public later:** you can flip the repo to public, publish to npm, and keep the history intact.

**Structure.** One workspace repo (a monorepo) with one package per design system:

```
ui-library/                 (private)
├─ packages/
│  ├─ tokens/               @kisenamiyuki/tokens (only if the systems share tokens)
│  ├─ retro-futurism/       @kisenamiyuki/retro-futurism
│  └─ pixel-flow/           @kisenamiyuki/pixel-flow
└─ apps/
   └─ storybook/            docs and workbench for all packages
```

One repo means one set of build, lint, test and release config, and one Storybook to browse every package. Each package is versioned and installed independently, so a project only pulls in the design system it uses.

**What happens to `design-systems`.** It stays public and keeps hosting the demo pages. The demo site can stay as static HTML and CSS, or later become a consumer of the published packages, which is a good real-world test. If it does, see the note on CI access under GitHub Packages below.

If privacy turns out not to matter, you can skip the new repo and restructure `design-systems` itself into this layout, with the Pages demos as one of the apps.

## Porting the design systems to React

Do this one system at a time. Start with Geometric Retro Futurism, which has less animation, and use it to settle the conventions before porting PIXEL/FLOW.

- **Inventory.** Go through each HTML page and list what becomes a component (buttons, panels, grids, the animated backgrounds) and what is only demo content. Demo layouts belong in Storybook stories, not in the package.
- **API design.** For each component, decide its props and variants. As a baseline, every component accepts `className` and passes through extra props to its root element, and interactive or measurable ones forward `ref`. Keep the API small; it's easy to add props and hard to remove them.
- **Tokens.** Extract colors, spacing, type and motion values into CSS variables with a per-system prefix (`--grf-*`, `--pf-*`) so the systems can't clash with each other or with the host app's variables. If both systems end up sharing values, that's what `packages/tokens` is for; otherwise skip it.
- **Animations.** Wrap canvas and `requestAnimationFrame` loops in components that:
  - hold the canvas in a `ref` and start the loop in `useEffect`, cancelling the frame and removing listeners in the cleanup;
  - pause when off-screen (`IntersectionObserver`) or when the tab is hidden (`visibilitychange`);
  - respect `prefers-reduced-motion` by rendering a static frame;
  - don't touch `window` or `document` during render, so they don't crash under server rendering.

  PIXEL/FLOW is already known to be CPU-heavy, so treat these as requirements, not nice-to-haves.
- **Styling approach.** Plain CSS files with prefixed class names (or CSS Modules) compiled into one stylesheet per package. Avoid runtime CSS-in-JS: it adds a dependency and complicates server rendering.
- **Assets.** Decide how fonts and images ship. Small images can be inlined by the build; fonts are better referenced from the stylesheet with `@font-face` and shipped in `dist/`, or documented as something the host app loads.

## What to set up in each package

This stays the same whichever way you install later.

- **Name and version.** `"name": "@kisenamiyuki/pixel-flow"`, `"version": "0.1.0"`. The scope must match your GitHub username, because GitHub Packages requires it. This does **not** reserve the scope on npmjs.com; if you want `@kisenamiyuki` there for going public later, create the npm user or org now.
- **One entry point.** `src/index.ts` exports only the public components. Anything not exported stays private.
- **React as a peer dependency.** List `react` and `react-dom` in `peerDependencies`, and also in `devDependencies` so the package's own build, tests and Storybook have a copy.
- **A build to `dist/`** with Vite library mode: ESM output, `.d.ts` types via `vite-plugin-dts` (Vite doesn't emit types on its own), and a single `styles.css`.
- **Tests.** Vitest with Testing Library. At minimum: each component renders, and animated components clean up their loops and listeners on unmount.

You could skip the build and ship raw TypeScript (Turborepo calls these "just-in-time packages"). That works inside a monorepo, but it gets fragile once other machines install the package, especially with CSS or Next.js. Building to `dist/` costs very little and makes publishing trivial.

### `package.json`

```json
{
  "name": "@kisenamiyuki/pixel-flow",
  "version": "0.1.0",
  "type": "module",
  "exports": {
    ".": {
      "types": "./dist/index.d.ts",
      "import": "./dist/index.js"
    },
    "./styles.css": "./dist/styles.css"
  },
  "files": ["dist"],
  "sideEffects": ["**/*.css"],
  "peerDependencies": {
    "react": "^18.0.0 || ^19.0.0",
    "react-dom": "^18.0.0 || ^19.0.0"
  },
  "repository": {
    "type": "git",
    "url": "git+https://github.com/kisenamiyuki/ui-library.git",
    "directory": "packages/pixel-flow"
  },
  "publishConfig": { "registry": "https://npm.pkg.github.com" }
}
```

- `exports` is the public boundary: consumers can import the components and the stylesheet (`import '@kisenamiyuki/pixel-flow/styles.css'`) and nothing else.
- `files` keeps source, tests and stories out of the published package.
- `sideEffects` tells bundlers the JS is safe to tree-shake but the CSS is not. Without the CSS entry, a consumer's bundler can silently drop the stylesheet import.
- `repository` is how GitHub Packages links the package to the repo.

### Vite config

```ts
// packages/pixel-flow/vite.config.ts
import { defineConfig } from 'vite'
import react from '@vitejs/plugin-react'
import dts from 'vite-plugin-dts'

export default defineConfig({
  plugins: [react(), dts({ include: ['src'] })],
  build: {
    lib: {
      entry: 'src/index.ts',
      formats: ['es'],
      fileName: 'index',
      cssFileName: 'styles', // Vite 6+; Vite 5 always emits style.css
    },
    rollupOptions: {
      // Never bundle React, including react/jsx-runtime,
      // or consumers get two copies and "Invalid hook call".
      external: [/^react(-dom)?($|\/)/],
    },
  },
})
```

**Next.js.** Components that use hooks or touch the DOM (all the animated ones) need `"use client"` at the top of the file to work in the App Router. Rollup strips this directive by default. Either build with `preserveModules` plus a plugin such as `rollup-plugin-preserve-directives`, or add the directive to the output another way, and check that it actually appears in `dist/`.

### Validate before publishing

Run these in CI on every change:

- `publint`: checks `package.json` fields and that `exports` point at files that exist.
- `@arethetypeswrong/cli` (`attw --pack`): checks that TypeScript resolves the types correctly for consumers.

They catch the mistakes that otherwise only show up after someone installs the package.

## Versioning and releases

- **Semver in 0.x.** While a package is at `0.x`, a minor bump is the breaking change: `^0.3.0` accepts `0.3.x` only and never `0.4.0`. Bump the minor for breaking changes and the patch for everything else until the API settles enough for `1.0.0`.
- **Changesets.** With several independently versioned packages, use Changesets. Each change adds a small changeset file; `changeset version` bumps the right packages, updates internal dependency ranges and writes each package's `CHANGELOG.md`; `changeset tag` creates per-package tags like `pixel-flow@0.1.0`. A bare `v0.1.0` tag can't say which package it refers to.
- **Publish with `bun publish`, not `npm publish`.** Inside the workspace, packages depend on each other with `workspace:*`. `bun publish` strips the `workspace:` protocol and writes the real version; `npm publish` ships the literal `workspace:*`, and installs fail. `changeset publish` runs `npm publish` under the hood and has the same problem, so use Changesets for versioning and tagging, and `bun publish` for the publish step. `bun publish` also honors `publishConfig.registry` and reads tokens from `.npmrc`.

## Using it on the same machine

- **Apps in the same repo:** use workspaces. Add `"workspaces": ["packages/*", "apps/*"]` to the root `package.json`, and in each app depend on `"@kisenamiyuki/pixel-flow": "workspace:*"`. This is the smoothest setup for day-to-day work.
- **Apps in other folders:** run `bun link` in `packages/pixel-flow`, then `bun link @kisenamiyuki/pixel-flow` in the app. A `"file:../path/to/pixel-flow"` dependency also works.
  - **The usual problem** is that the app ends up with two copies of React, and you get "Invalid hook call" errors. In the app's Vite config, set `resolve: { dedupe: ['react', 'react-dom'] }`. For webpack or Next.js, alias `react` and `react-dom` to the app's own copies.

## Using it on other machines: GitHub Packages

GitHub Packages is GitHub's package hosting. Its npm registry lives at `https://npm.pkg.github.com`, and a package published there can be linked to a repo and inherit its access rules, so a package from a private repo is only installable by you. You publish and install with version numbers, exactly as with npmjs.com, which makes it a rehearsal for publishing publicly later.

How it differs from npmjs.com:

- **Scoped names only.** Package names must be scoped to the owner: `@kisenamiyuki/pixel-flow`.
- **Registry per scope.** Your package manager has to be told to fetch that scope from GitHub (the `.npmrc` line below).
- **Authentication always.** Installing requires a token with `read:packages`, even for public packages. GitHub's docs state that GitHub Packages only supports authentication with a personal access token (classic), so fine-grained tokens won't work here.
- **Cost.** Public packages are free. Private packages share a free storage and transfer allowance (historically 500 MB storage and 1 GB transfer per month on the Free plan). Downloads inside GitHub Actions don't count toward transfer. A component library is a few hundred KB per version, so you won't get close.

### Setup

Each package already has `publishConfig.registry` set (see above). On every machine that installs, add an `.npmrc` to the consuming project (Bun reads `.npmrc` and expands `${VAR}` in it from the environment):

```
@kisenamiyuki:registry=https://npm.pkg.github.com
//npm.pkg.github.com/:_authToken=${GH_PACKAGES_TOKEN}
```

`GH_PACKAGES_TOKEN` is an environment variable holding your personal access token: `read:packages` on machines that only install, plus `write:packages` where you publish. The `.npmrc` itself contains no secret and can be committed.

Don't name the variable `GITHUB_TOKEN` on your own machines. `gh` and other tools read `GITHUB_TOKEN` and would use it in place of your normal login, so a token with only package scopes would break them.

Apps then depend on normal versions, like `"@kisenamiyuki/pixel-flow": "^0.3.0"`.

### Publishing from GitHub Actions

Publish from CI rather than from your machine: on a release commit (or tag), the workflow builds, runs the checks, and runs `bun publish` for each changed package.

- Give the workflow `permissions: { contents: write, packages: write }`.
- Map the built-in token to the same variable name the `.npmrc` uses: `env: { GH_PACKAGES_TOKEN: ${{ secrets.GITHUB_TOKEN }} }`. Then one `.npmrc` works locally and in CI.

### Installing from another repo's CI

If another repo's workflow installs the package (for example, the public `design-systems` demo site building with the packages), its `GITHUB_TOKEN` can only read the package if you grant that repo access in the package's settings on GitHub (Package settings → Manage Actions access). Without that, the install fails with a 403.

### Alternative: installing straight from git

You can skip the registry and depend on `"github:kisenamiyuki/ui-library#..."`, but package managers expect the package at the repo root, installing from a monorepo subfolder is only partly supported (pnpm can, npm can't, and Bun's docs don't mention it), and `dist/` would have to be committed because Bun doesn't run dependencies' install scripts by default. With a monorepo and GitHub Packages, there's no reason to use this.

## Making it public later

1. Make sure the `@kisenamiyuki` scope exists on npmjs.com (see above).
2. Add the things a public package needs: a `license` field and `LICENSE` file, and a README for each package.
3. In each package, replace `"publishConfig": { "registry": "https://npm.pkg.github.com" }` with `"publishConfig": { "access": "public" }`. Scoped packages are private by default on npm, so `access` is required.
4. Publish with `bun publish`, authenticated to npmjs.com.
5. Consumers delete the two GitHub lines from `.npmrc`. Component code and imports don't change.

Optionally flip the repo to public at the same time.

## Suggested path

1. **Set up the repo.** Create the private `ui-library` repo as a Bun workspace with the layout above, Storybook, Vitest, Changesets, and the Vite build config. Add `publint` and `attw` to CI.
2. **Extract tokens.** Pull Geometric Retro Futurism's colors, spacing, type and motion values into prefixed CSS variables.
3. **Port Geometric Retro Futurism.** Build its components with the API conventions above, with a story and a basic test for each.
4. **Publish `0.1.0`.** Build, validate, and publish `@kisenamiyuki/retro-futurism@0.1.0` to GitHub Packages from the Actions workflow.
5. **Install it on a second machine.** Install from GitHub Packages into a fresh Vite app, and if you use Next.js, into a Next.js app too. Fix whatever breaks before going further: this is where packaging mistakes show up.
6. **Port PIXEL/FLOW** using the same conventions, with extra attention to the animation rules: cleanup, pausing when off-screen and reduced motion.
