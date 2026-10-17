# Packaging a Private React Component Library

Goal: package a custom React component library for private use, both on the same machine and on other machines through a private GitHub repo, while keeping public publishing possible later.

The systematic part isn't the bundler or npm. It's turning the components into a real package with a clear boundary: a name, a version, a defined public API, and React listed as a peer dependency. Once you have that, "private now, public later" mostly comes down to where you install from. Publishing to npm later is about a two-line change.

## Copying JSX files vs. a package

Copying `.jsx` files into projects works, but it has problems:

- Each project ends up with its own drifting copy, and you can't tell which version it has.
- Nothing separates the public components from internal helpers, so projects start importing internals.
- Styles, tokens and dependencies aren't tied to the components, so you have to remember to copy them too.
- Updating means copying again by hand.

A package fixes all four: projects write `import { Button } from '@kisenamiyuki/ui'`, pin a version, and upgrade on purpose.

## What to set up now

Do this once, and it stays the same whichever way you install later:

- **The package.** Create `packages/ui` with `"name": "@kisenamiyuki/ui"` and `"version": "0.1.0"`. The scope should be your GitHub username, because GitHub Packages requires it and it reserves the name for later.
- **One entry point.** `src/index.ts` exports only the public components. Anything not exported stays private.
- **React as a peer dependency.** List `react` and `react-dom` in `peerDependencies`, not `dependencies`.
- **A build to `dist/`.** Use Vite library mode, producing ESM files, `.d.ts` types and a single `styles.css` (with your tokens as CSS variables).
- **Tags and a changelog.** Tag releases (`v0.1.0`) and keep a changelog. Changesets is optional at this size; hand-written tags plus a `CHANGELOG.md` are fine.
- **Storybook** (optional but useful): this is where you develop and browse components without needing a host app.

You could skip the build and ship raw TypeScript (Turborepo calls these "just-in-time packages"). That works inside a monorepo, but it gets fragile once other machines install the package, especially with CSS Modules or Next.js. Building to `dist/` costs very little and makes publishing later trivial.

## Using it on the same machine

- **Apps in the same repo:** use workspaces. Add `"workspaces": ["packages/*", "apps/*"]` to the root `package.json`, and in each app depend on `"@kisenamiyuki/ui": "workspace:*"`. Bun supports this. It's the smoothest setup for day-to-day work.
- **Apps in other folders:** run `bun link` in `packages/ui`, then `bun link @kisenamiyuki/ui` in the app. A `"file:../path/to/ui"` dependency also works.
  - **The usual problem** is that the app ends up with two copies of React, and you get "Invalid hook call" errors. In the app's Vite config, set `resolve: { dedupe: ['react', 'react-dom'] }`.

## What GitHub Packages is

GitHub Packages is GitHub's built-in package hosting. You can think of it as a package registry attached to your GitHub account, so you can publish packages next to your code instead of on the public npm registry.

**What it hosts.** It runs registries for several ecosystems: npm, container images (Docker, at `ghcr.io`), Maven/Gradle, NuGet and RubyGems. For a component library, the relevant one is the npm registry at `https://npm.pkg.github.com`.

**How it fits with your repos.** A package belongs to your account (or an organization) and can be linked to a repository. A package linked to a private repo can inherit that repo's access rules, so only you can install it. It also appears on the repo's page under "Packages".

**How it differs from npmjs.com:**

- **Name scope:** package names must be scoped to the owner, so `@kisenamiyuki/ui`, not just `ui`.
- **Telling your tools where to look:** your package manager has to send that scope to GitHub's registry. That's the `@kisenamiyuki:registry=...` line in `.npmrc`.
- **Authentication:** installing requires a personal access token with `read:packages`, even for public packages, unlike npmjs.com. Historically this has needed a classic token rather than a fine-grained one, so check GitHub's docs when you create it. Inside GitHub Actions, the built-in `GITHUB_TOKEN` handles auth for you.

**Cost.** Public packages are free. Private packages get a free allowance of storage and download bandwidth that depends on your plan; the Free plan has historically included 500 MB of storage and 1 GB of transfer per month. Downloads from inside GitHub Actions don't count toward the bandwidth limit. A component library is a few hundred KB per version, so you won't get close to the limits.

**Why it fits this use case.** You publish and install exactly as you would with real npm, using version numbers and `npm publish`. The difference is that it's private and lives next to your private repo. When you're ready to go public, you point the same package at npmjs.com instead.

## Using it on other machines from a private GitHub repo

There are two good options.

### Option 1: GitHub Packages, a private npm registry (recommended)

This is the closest to real publishing, so it rehearses what you'd do later. It's free for private packages within the storage quota.

In the library's `package.json`:

```json
{
  "name": "@kisenamiyuki/ui",
  "publishConfig": { "registry": "https://npm.pkg.github.com" }
}
```

On each machine that installs it, add an `.npmrc` (or Bun's equivalent in `bunfig.toml`):

```
@kisenamiyuki:registry=https://npm.pkg.github.com
//npm.pkg.github.com/:_authToken=${GITHUB_TOKEN}
```

`GITHUB_TOKEN` is a personal access token with `read:packages` (and `write:packages` on the machine that publishes). Each app then depends on a normal version like `"@kisenamiyuki/ui": "^0.3.0"`. A small GitHub Action can publish whenever you push a tag.

### Option 2: Install straight from git

This needs no registry: `"@kisenamiyuki/ui": "github:kisenamiyuki/ui#v0.3.0"`. It works for private repos as long as the machine has SSH or `gh` access. It has two catches:

- Package managers expect the package to sit at the repo root. Installing from a subfolder of a monorepo is only partly supported (pnpm can, npm can't, and Bun's support is unverified). So you'd want the library in its own repo.
- The built `dist/` has to exist when the app installs it. Either commit `dist/` on tagged releases, or rely on a `prepare` script. Bun doesn't run dependencies' install scripts by default, so committing `dist/` is safer.

## Making it public later

- **From GitHub Packages:** remove the `publishConfig.registry` line, add `"publishConfig": { "access": "public" }`, and run `npm publish`. Consumers delete one line from `.npmrc`. Component code and imports don't change.
- **From a git dependency:** consumers switch from the `github:` URL to a version number.

## Repo strategy: use a dedicated private repo

Make a dedicated **private** repo for the library and keep `design-systems` as the public showcase.

**The main reason is visibility.** GitHub sets public or private per repository, so you can't have a private folder inside a public repo. `design-systems` has to stay public for the demo pages: GitHub Pages on a private repo needs a paid plan. So a private library needs its own repo.

**It also helps for other reasons:**

- **Linked package:** a package published to GitHub Packages links to its repo and can inherit that repo's access rules. A dedicated private repo keeps that simple.
- **Separate history:** library releases, tags and the changelog stay apart from demo-site changes, so `v0.3.0` means "the library at version 0.3.0" and nothing else.
- **Git installs:** if you ever install from git instead of GitHub Packages, package managers expect the package at the repo root, or at least in a predictable workspace. A dedicated repo avoids the subfolder problem.
- **Going public later:** you can flip the repo to public, publish to npm, and keep the history intact.

**How to structure the new repo.** Make it a single workspace repo (a monorepo) rather than one repo per design system. For example:

```
ui-library/                 (private)
├─ packages/
│  ├─ tokens/               shared tokens, if the systems share any
│  ├─ retro-futurism/       @kisenamiyuki/retro-futurism
│  └─ pixel-flow/           @kisenamiyuki/pixel-flow
└─ apps/
   └─ storybook/            docs and workbench for all packages
```

One repo means one set of build, lint and release config, and one Storybook to browse every package. Each package still gets its own version, so you can install only the design system a project needs.

**What happens to `design-systems`.** It stays public and keeps hosting the demo pages. Once the library exists, the demo site can either stay as-is (static HTML and CSS) or become a consumer of the published packages. That second option is also a good real-world test of the library.

The one case where a new repo isn't needed is if privacy doesn't matter after all. Then you could restructure `design-systems` itself into the workspace layout above, with the Pages demos as one of the apps.

## Suggested path

1. Create a private repo set up as a Bun workspace, with one package per design system under `packages/` and a Storybook that uses them.
2. Build with Vite library mode and tag `v0.1.0`.
3. Publish to GitHub Packages from a tag-triggered Action, and install it on your other machines with a token.
