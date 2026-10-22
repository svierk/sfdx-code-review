# 🔎 SFDX Code Review

This repository implements a unified GitHub composite action for the **static code review** of Salesforce projects. It bundles the checks that are otherwise scattered across separate CI steps into a single, configurable quality gate:

- 🛡️ **Salesforce Code Analyzer** — static analysis for Apex, LWC, Aura and more (PMD, ESLint, RetireJS, Flow, …)
- 🎨 **Prettier** — code formatting check
- 📏 **ESLint** — JavaScript/LWC code quality, with a native report for SonarQube/SonarCloud

The Salesforce Code Analyzer is enabled by default; Prettier and ESLint are opt-in.

> **Why all three?** The Salesforce Code Analyzer does *not* check formatting (Prettier is not a static-analysis engine), and while it bundles an ESLint engine, it emits its own unified format rather than the native `eslint-report.json` that Sonar's `sonar.eslint.reportPaths` consumes. Running ESLint and Prettier here keeps everything in one gate and produces Sonar-ready reports.

## Usage

### Full code review (analyzer + formatting + linting)

```yaml
jobs:
  code-review:
    name: Code Review
    runs-on: ubuntu-latest
    permissions:
      contents: read # least-privilege GITHUB_TOKEN - this job only reads the repository
    steps:
      - name: Checkout
        uses: actions/checkout@v7.0.1
        with:
          fetch-depth: 0 # full history, required for the Sonar analysis
          persist-credentials: false # the GITHUB_TOKEN is not needed after the checkout - don't leave it in .git/config

      - name: Select Node Version
        uses: svierk/get-node-version@v1.5.1

      - name: Install Dependencies
        run: npm ci --ignore-scripts

      - name: Install SF CLI
        uses: svierk/sfdx-cli-setup@v1.1.3

      - name: Code Review
        uses: svierk/sfdx-code-review@v1.1.0
        with:
          code-analyzer: true
          prettier: true
          eslint: true
          workspace: force-app
          severity-threshold: 3

      - name: SonarCloud Scan
        uses: SonarSource/sonarqube-scan-action@v8.2.1
        env:
          SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }} # SONAR_HOST_URL is additionally required for a self-hosted SonarQube Server
```

> Prettier and ESLint run locally via Node, so install the project dependencies first (e.g. `npm ci`) and make sure the project provides the matching config (`.prettierrc`, `eslint.config.js`/`.eslintrc`).

> **Applying your ESLint config to the Code Analyzer engine.** By default the Code Analyzer's bundled ESLint engine uses its own base rules and does *not* apply your workspace `eslint.config.js` — it logs a hint when it finds one (`ESLint configuration file 'eslint.config.js' was found but not applied`). This is expected: the separate ESLint check above (`eslint: true`) is what applies your project config and produces the Sonar report. If you also want your config applied *inside* the Code Analyzer's ESLint engine, this is an engine setting (no CLI flag exists), so pass a `code-analyzer.yml` via the `config-file` input:
>
> ```yaml
> engines:
>   eslint:
>     # either point at the config explicitly …
>     eslint_config_file: eslint.config.js
>     # … or let the engine discover it automatically
>     auto_discover_eslint_config: true
> ```

### List inputs (comma-separated or multi-line)

The list inputs - `workspace`, `rule-selector` and `output-file` - accept a comma-separated string as well as a multi-line block, and the two notations can be mixed freely within one step. A comma-separated string keeps short lists on a single line, a block stays readable once a list grows:

```yaml
      - name: Code Review
        uses: svierk/sfdx-code-review@v1.1.0
        with:
          rule-selector: 'Security,Performance' # comma-separated
          workspace: | # multi-line block
            force-app
            my-other-app
          output-file: |
            results.html
            results.sarif
```

Surrounding whitespace and blank lines are ignored, so `'Security, Performance'` is read exactly like the tightly packed `'Security,Performance'`. Whitespace inside a value is kept, e.g. for a path like `force-app/main/my folder`.

### Uploading Code Analyzer results to GitHub code scanning

When producing a SARIF file, let the analyzer report findings without failing the job immediately, then upload the SARIF. Uploading to code scanning requires the job to additionally grant `security-events: write`:

```yaml
    permissions:
      contents: read
      security-events: write # required to upload the SARIF file to GitHub code scanning

    steps:
      - name: Code Review
        continue-on-error: true
        uses: svierk/sfdx-code-review@v1.1.0
        with:
          workspace: force-app
          output-file: results.sarif
          severity-threshold: 3

      - name: Upload SARIF
        uses: github/codeql-action/upload-sarif@v4.37.7
        with:
          sarif_file: results.sarif
```

The Code Analyzer only writes a SARIF run for each engine with violations. The action completes every `.sarif` output file with an empty run for each engine that ran without violations, so the upload also succeeds when no violations are found, and code scanning closes the alerts of engines that no longer report any.

## Inputs

| Name                 | Required | Default                                                | Description                                                                                   |
| -------------------- | -------- | ------------------------------------------------------ | --------------------------------------------------------------------------------------------- |
| `code-analyzer`      | no       | `true`                                                 | Run the Salesforce Code Analyzer.                                                              |
| `prettier`           | no       | `false`                                                | Run a Prettier formatting check.                                                              |
| `eslint`             | no       | `false`                                                | Run ESLint and produce a report (e.g. for Sonar).                                             |
| `workspace`          | no       |                                                        | Files/folders for the Code Analyzer, comma-separated or multi-line. Defaults to the current directory. |
| `rule-selector`      | no       | `Recommended`                                          | Code Analyzer rule selectors, comma-separated or multi-line, e.g. `Security,Performance`.      |
| `config-file`        | no       |                                                        | Path to a Code Analyzer configuration file (`code-analyzer.yml`).                             |
| `output-file`        | no       |                                                        | Code Analyzer output files, comma-separated or multi-line; format derived from the extension (`.sarif`, `.html`, `.json`, …). |
| `severity-threshold` | no       |                                                        | Fail if the Code Analyzer finds a violation with this severity or higher (`1`–`5` / `Critical`–`Info`). |
| `install-plugin`     | no       | `true`                                                 | Install the Code Analyzer plugin before running. Set to `false` if installed via sfdx-cli-setup. |
| `plugin-version`     | no       |                                                        | Specific version of the `code-analyzer` plugin to install, e.g. `5.0.0`.                      |
| `prettier-command`   | no       | `npx prettier --check .`                               | Command used to check formatting. Override to use your own, e.g. `npm run prettier:verify`.    |
| `eslint-command`     | no       | `npx eslint . --format json --output-file eslint-report.json` | Command used to run ESLint and write its report. Override to use your own, e.g. `npm run lint:sonar`. |
| `eslint-report-path` | no       | `./eslint-report.json`                                 | Path to the ESLint JSON report, used to report error/warning counts.                          |
| `step-summary`       | no       | `true`                                                 | Write a result section to the GitHub Actions [job summary](https://docs.github.com/en/actions/using-workflows/workflow-commands-for-github-actions#adding-a-job-summary). Set to `false` to avoid collisions with a custom workflow summary. |

## Outputs

| Name                    | Description                                              |
| ----------------------- | ------------------------------------------------------- |
| `code-analyzer-outcome` | Outcome of the Code Analyzer run (`Passed`/`Failed`).   |
| `prettier-outcome`      | Outcome of the Prettier check (`Passed`/`Failed`).      |
| `eslint-outcome`        | Outcome of the ESLint run (`Passed`/`Failed`).          |
| `eslint-errors`         | Number of ESLint errors.                                |
| `eslint-warnings`       | Number of ESLint warnings.                              |

Outputs for a check that was not run are empty. The step fails if any enabled check fails, while still writing the summary and outputs.

## Sonar integration

ESLint writes its native JSON report so Sonar can pick it up directly:

```properties
sonar.eslint.reportPaths=./eslint-report.json
```

The Salesforce Code Analyzer can additionally emit a SARIF/JSON file via `output-file` for GitHub code scanning or other tools.

The scan step itself only needs `SONAR_TOKEN` as an environment variable (plus `SONAR_HOST_URL` for a self-hosted SonarQube Server). A `GITHUB_TOKEN` is **not** required: pull request decoration is performed by the SonarQube Cloud GitHub App, not by the scanner, so passing the token would hand a credential to a third-party action for no gain.

## Security & versioning

Every `uses:` reference in the snippets above is **pinned to an exact release version**, e.g. `svierk/sfdx-code-review@v1.1.0`. Do the same in your own pipelines:

- **Never reference a mutable ref** such as `@main`, `@master` or `@v1`. It runs whatever code sits on that branch/tag at run time — with access to your repository and org credentials — so a compromised or rewritten ref would run unnoticed.
- **Good — pin to an exact release tag** (`@v1.0.0`). Readable, concrete, and bumped through reviewed pull requests.
- **Strictest — pin to a full-length commit SHA** (`@a1b2c3d…`) with the version as a trailing comment. A SHA can never be re-pointed by the publisher; the cost is readability.
- **Enable [Dependabot](https://docs.github.com/en/code-security/dependabot/working-with-dependabot/keeping-your-actions-up-to-date-with-dependabot) for `github-actions`** in the repository that consumes this action, so those pins are bumped for you instead of silently ageing. Note that Dependabot only scans `.github/workflows` — copy-paste snippets in documentation have to be updated by hand.

This action itself needs **no secrets and no `GITHUB_TOKEN` permissions** — it only runs the Salesforce Code Analyzer, Prettier and ESLint against the checked-out sources. Grant the calling job the least privilege it needs (`contents: read` is enough, plus `security-events: write` when uploading SARIF) and use `persist-credentials: false` on the checkout so the token is not written to `.git/config`, where the CLI, a plugin or an ESLint config could pick it up.

If you pass credentials to *other* steps of the same workflow, reference them as **environment variables** (`env: SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}`, then `"$SONAR_TOKEN"` in the script), never by interpolating `${{ secrets.… }}` into a `run:` script itself — that would allow command injection and can leak values into the log. The steps of this action follow that rule for all of their inputs: every input is passed through `env:` and referenced as a shell variable, so a value such as `eslint-command` or `workspace` can never be expanded into the script body.

## References

- [Salesforce Code Analyzer](https://developer.salesforce.com/docs/platform/salesforce-code-analyzer/overview) | Documentation
- [code-analyzer run](https://developer.salesforce.com/docs/platform/salesforce-code-analyzer/references/code-analyzer-command-reference) | Command Reference
- [Prettier](https://prettier.io/docs/en/cli.html) | CLI
- [ESLint](https://eslint.org/docs/latest/use/command-line-interface) | CLI

## Releases

Latest release notes can be found on the [release page](https://github.com/svierk/sfdx-code-review/releases).

## License

The scripts and documentation in this project are released under the [MIT License](https://github.com/svierk/sfdx-code-review/blob/main/LICENSE).
