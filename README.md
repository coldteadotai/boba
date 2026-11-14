# Boba

Run your Playwright end-to-end suite on [Coldtea Boba](https://www.coldtea.ai/boba) and get one check on your commit or pull request, with each failure annotated on the line that failed.

Boba runs your tests on copies of your own GitHub Actions runner, taken after your workflow's earlier steps (checkout, install, build, starting your app). Later pushes reuse that setup, so they skip straight to the tests. Turbo spreads a suite across several machines.

## Before you start

1. Add Boba to GitHub and add a card at [coldtea.ai/boba/install](https://www.coldtea.ai/boba/install). Your jobs only run on Boba once the app is installed and a card is on file.
2. Find the workflow job that runs your Playwright tests.

## Usage

The change is two lines: the job's `runs-on`, and the step that runs Playwright.

```yaml
jobs:
  e2e:
    runs-on: coldtea-boba # was: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      # ...your install, build and app start steps, unchanged
      - uses: coldteadotai/boba@v1 # was: run: npx playwright test
```

- Keep every step before the Playwright step as it is, and leave your Playwright config as it is.
- If the old test step also started your app, an API or a worker in the background, move those commands into their own step just before `coldteadotai/boba@v1`. Boba starts the tests with them already running. A new step starts a fresh shell, so write any variable they export to `$GITHUB_ENV`.
- Boba stops a job after 60 minutes, so keep each Boba job under that.

### Pull requests from forks

If your repository takes pull requests from forks, give them runners of their own, so they start warm without ever reading a runner your own branches saved:

```yaml
runs-on: ${{ github.event_name == 'pull_request' && github.event.pull_request.head.repo.full_name != github.repository && 'coldtea-boba-outside' || 'coldtea-boba' }}
```

### Cancel superseded runs

Add a concurrency block, so a newer push to a branch or pull request cancels the run it replaces, while every commit on the default branch and in a merge queue keeps its own run:

```yaml
concurrency:
  group: boba-${{ github.workflow }}-${{ (github.event_name == 'merge_group' || github.ref == format('refs/heads/{0}', github.event.repository.default_branch)) && github.sha || github.ref }}
  cancel-in-progress: true
```

### Turbo

```yaml
      - uses: coldteadotai/boba@v1
        with:
          turbo: true
```

Turbo spreads the suite across several machines. It costs about 3x Standard per run.

### Several Playwright steps in one job

Put them all in one Boba step: its own `config` and `playwright-args` run as one suite, and each other step goes under `suites`, with the config, args and env it used. Boba runs them all at once and posts one check with a section each.

```yaml
      - uses: coldteadotai/boba@v1
        with:
          suites: |
            - name: email
              config: e2e/playwright.config.email.ts
              args: --grep @email
              env:
                E2E_ALLOW_EMAIL_CHANGE: 'false'
```

## Inputs

On `runs-on: coldtea-boba`, every input is optional.

| Input | Default | What it does |
| --- | --- | --- |
| `working-directory` | | The folder holding the suite's `playwright.config`, relative to the checkout. Only needed when the repository has several configs. |
| `config` | | The Playwright config file, when it isn't named `playwright.config` (what you pass to `playwright test -c`). |
| `project` | | The Playwright project to run. Empty runs every project. |
| `playwright-args` | | Extra arguments for `playwright test`, separated by spaces. |
| `turbo` | `false` | `true` spreads the suite across several machines; `false` runs it on one. |
| `fail-fast` | `true` | On a pull request, stop at the first test that still fails after its retries; the check says how many tests did not run. Pushes and merge queues always run the whole suite. |
| `suites` | | Only for a job with more than one Playwright step: those steps' runs, as a YAML list (see above). |
| `timeout-minutes` | `60` | How long the step waits for results. |
| `check-name` | `Boba` | Name of the check run. |
| `post` | `true` | `false` prints the check instead of posting it. |
| `job-id` | | Attach to a Boba job that already exists instead of starting one. |
| `api-url` | | Boba's address. Not needed on `coldtea-boba`, where the runner knows it. |
| `api-token`, `secrets`, `github-token` | | Not needed on `coldtea-boba`. Kept for workflows from before Boba's runners. |

## What you get

- **One check** on the commit, named `Boba` by default. A failing test is annotated on the line that failed, and failures are grouped by cause. On a pull request, a test that already fails on your default branch is marked as such, so it doesn't read as broken by your change.
- **A test report** with traces. A public repository's report opens for anyone with the link. A private repository's opens after GitHub sign-in, for people who can read the repository.
- **A second check, "Boba saved VM"**, when Boba could not keep a copy of your runner for the next push, saying why and how to fix it.

## Security

Your secrets stay out of saved runners and reports: each is scanned before it is kept or published, and withheld when a secret is found. See [how Boba handles your code and secrets](https://www.coldtea.ai/docs/platform/teams-security/security-and-privacy#boba-ci-secrets-saved-runners-and-reports).

## License

[MIT](LICENSE)
