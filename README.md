# Boba

Run your Playwright end-to-end suite on [Coldtea Boba](https://www.coldtea.ai/boba) and get one check on your commit or pull request, with each failure annotated on the line that failed.

On your first run, Boba takes about as long as your CI does now. On every push after that, Boba gets to your tests faster. Turn on Turbo when you want the fastest result.

Read the full docs at [coldtea.ai/boba/docs](https://www.coldtea.ai/boba/docs).

## Before you start

1. Add Boba to GitHub and add a card at [coldtea.ai/boba/install](https://www.coldtea.ai/boba/install). Boba runs no job until you install the app and add a card.
2. Find the workflow job that runs your Playwright tests.

## Usage

Change two lines: the job's `runs-on`, and the step that runs Playwright.

```yaml
jobs:
  e2e:
    runs-on: coldtea-boba # was: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      # ...your install, build and app start steps, unchanged
      - uses: coldteadotai/boba@v1 # was: run: npx playwright test
```

- Keep every step before the Playwright step as it is, and leave your Playwright config alone.
- If the old test step also started your app, an API or a worker in the background, move those commands into their own step just before `coldteadotai/boba@v1`. Boba starts the tests with them already running. GitHub starts each step in a fresh shell, so write any variable they export to `$GITHUB_ENV`.
- Boba stops a job after 60 minutes. Keep each Boba job under that.

### Pull requests from forks

If your repository takes pull requests from forks, send them to the `coldtea-boba-outside` label, so Boba never runs fork code on a runner your own branches use:

```yaml
runs-on: ${{ github.event_name == 'pull_request' && github.event.pull_request.head.repo.full_name != github.repository && 'coldtea-boba-outside' || 'coldtea-boba' }}
```

### Cancel superseded runs

Add this concurrency block and GitHub cancels a run when you push again to the same branch or pull request. GitHub still runs every commit on your default branch and in a merge queue.

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

Turbo gets you the fastest result. A Turbo run costs about 3x a Standard run. See [Billing](https://www.coldtea.ai/boba/docs/billing).

### Several Playwright steps in one job

Put them all in one Boba step. Boba runs the step's own `config` and `playwright-args` as one suite. List each other step under `suites`, with the config, args and env it used. Boba runs them all and posts one check with a section for each.

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

On `runs-on: coldtea-boba` you can leave out every input.

| Input | Default | What it does |
| --- | --- | --- |
| `working-directory` | | The folder that holds the suite's `playwright.config`, relative to the checkout. Set it only when your repository has several configs. |
| `config` | | The Playwright config file, when you named it something other than `playwright.config`. You pass the same file to `playwright test -c`. |
| `project` | | The Playwright project to run. Leave it empty and Boba runs every project. |
| `playwright-args` | | Extra arguments Boba passes to `playwright test`. Separate them with spaces. |
| `turbo` | `false` | `true` runs the suite on Turbo, the fastest tier. `false` runs it on Standard, the lowest price per run. |
| `fail-fast` | `true` | On a pull request, Boba stops at the first test that still fails after its retries, and the check says how many tests didn't run. On pushes and merge queues Boba always runs the whole suite. |
| `suites` | | Only for a job with more than one Playwright step: list those steps' runs here, as a YAML list (see above). |
| `timeout-minutes` | `60` | How many minutes the step waits for results. |
| `check-name` | `Boba` | The name Boba gives the check run. |
| `post` | `true` | Set `false` and Boba prints the check instead of posting it. |
| `job-id` | | Attaches the step to a Boba job that already exists, so Boba doesn't start a new one. |
| `api-url` | | Boba's address. You don't need it on `coldtea-boba`, where the runner knows it. |
| `api-token`, `secrets`, `github-token` | | You don't need these on `coldtea-boba`. Boba keeps them for workflows written before Boba had its own runners. |

## What you get

- **One check** on the commit, named `Boba` unless you set `check-name`. Boba annotates each failing test on the line that failed and groups failures by cause. On a pull request, Boba marks a test that already fails on your default branch, so it doesn't read as broken by your change.
- **A test report** with traces. Anyone with the link can open a public repository's report. For a private repository, you sign in with GitHub, and Boba opens the report if you can read the repository.
- **A second check, "Boba saved VM"**, when your job left something behind for you to fix, such as a secret. It says where and how to fix it. When it's a secret, Boba runs your next push like a first run.

Look up any check message in [Troubleshooting](https://www.coldtea.ai/boba/docs/troubleshooting) to see what to do.

## Security

Boba keeps your secrets out of everything it stores and publishes, and withholds a report that shows one. Boba keeps your data in the EU. Read [what Boba keeps, who can read it and when Boba deletes it](https://www.coldtea.ai/boba/docs/security).

## License

[MIT](LICENSE)
