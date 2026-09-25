# node-github-actions

A starter CI/CD pipeline for a Node.js app on GitHub Actions. On every change it tests the app, builds a Docker image, and then deploys. Tests run in parallel, and npm and Docker layers are cached.

> **Status:** This is a template. The build, test and deploy steps only print placeholder messages for now. Replace them with real commands as the app grows (see [Customizing](#customizing)).

## Pipeline

The workflow is in [`.github/workflows/ci.yml`](.github/workflows/ci.yml).

```
             ┌──────────────────┐
         ┌──▶│ unit tests       │──┐
         │   └──────────────────┘  │
 push /  │   ┌──────────────────┐  │   ┌──────────┐
 PR  ────┼──▶│ integration tests│──┼──▶│  deploy  │  (main only)
         │   └──────────────────┘  │   └──────────┘
         │   ┌──────────────────┐  │
         └──▶│ docker image     │──┘
             └──────────────────┘
```

| Job | What it does | Runs on |
| --- | --- | --- |
| `test` (matrix: `unit`, `integration`) | Installs dependencies with a cached npm store, builds, and runs each test suite on its own runner | Every push and PR to `main` |
| `image` | Builds the Docker image with Buildx, using layers cached in the GitHub Actions cache, and pushes it to GHCR. Skipped if there's no `Dockerfile` | Every push and PR; **pushes** only on `main` |
| `deploy` | Deploys the image, tagged with the commit SHA, to the `production` environment | Pushes to `main`, after `test` and `image` pass |

### Triggers

- Pushes and pull requests to `main`
- Manual runs from the **Actions** tab (`workflow_dispatch`)
- Changes that only touch `*.md`, `docs/**` or `.idea/**` **don't** trigger a run
- A new push to the same branch cancels the run that's still in progress

### Performance and caching

- The test suites and the image build run at the same time
- The npm download cache (`~/.npm`) is keyed on `package-lock.json` or `package.json`, so it works without a lockfile
- Docker layers are cached with `type=gha,mode=max`, so layers that haven't changed are reused
- Every job has a `timeout-minutes`, so a stuck run can't hold a runner for long

## Getting started

Requirements: Node.js 20 or later, and npm.

```bash
git clone <your-repo-url>
cd node-github-actions
npm install
npm test
```

Commit the `package-lock.json` that `npm install` generates. With a lockfile in the repo, CI uses `npm ci`, which gives the same install every time.

## Setup on GitHub

1. **Container registry:** no setup needed. Images are pushed to `ghcr.io/<owner>/<repo>` with the built-in `GITHUB_TOKEN`.
2. **Production environment:** in **Settings → Environments**, create a `production` environment. You can add required reviewers or restrict deployment branches there.
3. **Deployment credentials:** when you replace the placeholder deploy step, add the secrets it needs (for example a kubeconfig) under **Settings → Secrets and variables → Actions**.

## Customizing

| Placeholder step | Replace with, for example |
| --- | --- |
| Build the application | `npm run build` |
| Run unit tests | `npm run test:unit` |
| Run integration tests | `npm run test:integration` |
| Deploy the application | `kubectl set image deployment/app app=ghcr.io/<owner>/<repo>:${{ needs.image.outputs.tag }}` |

Also:

- Replace the placeholder `test` script in `package.json`, which always fails, with a real test runner, such as `node --test`
- Add a `Dockerfile` in the repo root to turn on the image build
- Change the Node.js version in the `NODE_VERSION` env var at the top of the workflow

## License

ISC
