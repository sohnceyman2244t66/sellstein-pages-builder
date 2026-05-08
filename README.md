# sellstein-pages-builder

Build worker for the [SellStein](https://sellstein.com) Custom Website Builder.

This repo runs no application code. It exists solely as a target for
`repository_dispatch` events sent from `operator-api`. When a merchant clicks
**Deploy** on a build-required project (Next.js, Astro, SvelteKit, Vite +
React, Vue, etc.), the API:

1. Tarballs the project source from R2.
2. Generates a one-shot signed URL pointing at that tarball.
3. Generates a one-shot HMAC callback token.
4. Sends a `repository_dispatch` event with type `build-deploy` carrying the
   source URL, callback URL, callback token, framework, and build command.

The workflow in `.github/workflows/build-deploy.yml` then downloads the
source, runs `npm install` + the framework's build command, tars the build
output, and POSTs it back to operator-api. Operator-api forwards the build
output to Cloudflare Pages Direct Upload and returns the resulting
`*.pages.dev` URL to the merchant via the deploy stream.

## Do not push directly

This repo is driven by `repository_dispatch` events. There is nothing to
deploy from here. The only file that matters is
`.github/workflows/build-deploy.yml`. Edits to that workflow ship by pushing
to `main` (subsequent dispatches will use the new workflow).

## Why GitHub Actions?

CF Pages' Git integration requires a CF↔GitHub OAuth handshake per merchant.
Direct Upload via Actions sidesteps that — operator-api owns the source code,
operator-api owns the upload, and GitHub Actions is purely the build runner.
Free tier ample for the platform's expected build volume.

## Supported runtimes

The dispatched payload sets `runtime` to one of:

| `runtime` | Frameworks |
|-----------|------------|
| `node`    | Vite + React/Vue/Svelte/Solid/Preact/Lit, Next.js, Nuxt, SvelteKit, Astro, Eleventy, Docusaurus, VitePress, VuePress, Storybook, Webpack/Parcel, Hexo, Stencil, Qwik, Remix, Gatsby |
| `hugo`    | Hugo |
| `jekyll`  | Jekyll |
| `python`  | MkDocs, Pelican |

The workflow installs the appropriate runtime per dispatch.

## Security

- The source URL is signed by operator-api with a short TTL.
- The callback URL accepts the build output ONLY when the bearer token
  validates against operator-api's HMAC secret.
- Tokens are single-use — operator-api marks them consumed on first valid
  POST and rejects replays.
- Workflows run in ephemeral runners with no access to merchant data
  outside the dispatched source.

## Maintenance

- Update Node version pin in the workflow when Node 22 LTS is the default.
- Add new runtimes by extending the `Set up …` steps and the `case` in
  the `Install dependencies` step.
