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

The current `design-systems` repo looks public (it hosts the GitHub Pages demos). So either make a new private repo for the library, or keep the library private and let the demo site use it as a consumer.

## Suggested path

1. Restructure into a Bun workspace with `packages/ui` and an app or Storybook that uses it.
2. Build with Vite library mode and tag `v0.1.0`.
3. Publish to GitHub Packages from a tag-triggered Action, and install it on your other machines with a token.
