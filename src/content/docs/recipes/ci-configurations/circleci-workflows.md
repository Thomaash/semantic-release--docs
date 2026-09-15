---
title: "CircleCI 2.0"
description: "Set up semantic-release on CircleCI with OIDC trusted publishing (recommended) and `NPM_TOKEN` fallback."
---

Use this recipe to run semantic-release on CircleCI with a secure default setup. It covers npm trusted publishing with OIDC (recommended), `NPM_TOKEN` fallback, and a verify-then-release workflow.

## Quick start

1. Choose an npm publishing path:
   - **Recommended:** OIDC trusted publishing. Follow [Trusted publishing (OIDC) on CircleCI](#trusted-publishing-oidc-on-circleci).
   - **Fallback:** `NPM_TOKEN` only when OIDC cannot be used.
2. Configure `.circleci/config.yml` using the [example workflow](#circleciconfigyml-configuration-for-node-projects).
3. Push to your primary release branch (`main`/`master`) and verify with the [Readiness and pitfalls checklist](#readiness-and-pitfalls-checklist).

## Environment variables

The [Authentication](/usage/ci-configuration#authentication) environment variables can be configured in [CircleCI project settings and contexts](https://circleci.com/docs/guides/security/contexts/).

- `NPM_ID_TOKEN` (recommended path): generated in the release step from CircleCI OIDC and exported before running semantic-release.
- `NPM_TOKEN` (fallback only): use only if OIDC trusted publishing is unavailable for your setup.
- `GH_TOKEN` / `GITHUB_TOKEN`: still required for GitHub release features (for example GitHub releases, release notes, and issue/PR comments) when those plugins are used.

## Trusted publishing (OIDC) on CircleCI

Use CircleCI's official guide: [Publish to npm using OIDC trusted publishing](https://circleci.com/docs/guides/deploy/deploy-to-npm-registry/).

With CircleCI trusted publishing configured on npm, semantic-release requires no special plugin configuration beyond your normal release setup. semantic-release detects CircleCI and uses `NPM_ID_TOKEN` to obtain a short-lived npm publish token.

CircleCI's guide currently states npm provenance attestations are **not** supported yet for packages published from CircleCI via trusted publishing.

You can optionally protect release credentials and execution with a restricted CircleCI context (for example project/branch restrictions and no-SSH-rerun). Keep setup details in CircleCI's official guide.

## Multiple Node jobs configuration

### `.circleci/config.yml` configuration for Node projects

This example keeps verification before release and uses OIDC trusted publishing in the release job. It scopes release execution to the primary release branch.

:::note
The release job must satisfy npm trusted-publishing prerequisites: Node.js >= 22.14.0 and npm >= 11.5.1.
:::

```yaml
version: 2.1
orbs:
  node: circleci/node@5.0.0

jobs:
  release:
    docker:
      - image: cimg/node:22.14.0
    steps:
      - checkout
      - node/install-packages # Install and automatically cache packages
      - run:
          name: Ensure npm trusted-publishing prerequisites
          command: npm install --global npm@11.5.1
      # Run optional required steps before releasing
      # - run: npm run build-script
      - run:
          name: Release
          command: |
            export NPM_ID_TOKEN=$(circleci run oidc get --claims '{"aud": "npm:registry.npmjs.org"}')
            npx semantic-release

workflows:
  test_and_release:
    # Run tests first, then release only when tests are successful
    jobs:
      - node/test:
          matrix:
            parameters:
              version:
                - 20.19.0
                - 22.14.0
      - release:
          # Optional: use a protected context for release credentials/settings
          # context: release-protected
          filters:
            branches:
              only:
                - main # or master
          requires:
            - node/test
```

## Readiness and pitfalls checklist

- Your workflow triggers releases from your primary release branch (`main` or `master`).
- Your npm Trusted Publisher for CircleCI is configured with the correct required CircleCI identifiers.
- The OIDC token uses the required custom audience claim: `npm:registry.npmjs.org`.
- The release executor meets prerequisites (Node.js >= 22.14.0 and npm >= 11.5.1).
- If you use a CircleCI context for release, protect it with appropriate project/branch restrictions (and optional no-SSH-rerun controls).
- `GH_TOKEN` / `GITHUB_TOKEN` is available when using GitHub release functionality.
- Use `NPM_TOKEN` only as a fallback if OIDC trusted publishing cannot be used.
