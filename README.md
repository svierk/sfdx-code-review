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
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Select Node Version
        uses: svierk/get-node-version@main

      - name: Install Dependencies
        run: npm install --ignore-scripts

      - name: Install SF CLI
        uses: svierk/sfdx-cli-setup@main

      - name: Code Review
        uses: svierk/sfdx-code-review@main
        with:
          code-analyzer: true
          prettier: true
          eslint: true
          workspace: force-app
          severity-threshold: 3

      - name: SonarCloud Scan
        uses: SonarSource/sonarqube-scan-action@master
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
```

> Prettier and ESLint run locally via Node, so install the project dependencies first (e.g. `npm install`) and make sure the project provides the matching config (`.prettierrc`, `eslint.config.js`/`.eslintrc`).

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

### Uploading Code Analyzer results to GitHub code scanning

When producing a SARIF file, let the analyzer report findings without failing the job immediately, then upload the SARIF:

```yaml
      - name: Code Review
        continue-on-error: true
        uses: svierk/sfdx-code-review@main
        with:
          workspace: force-app
          output-file: results.sarif
          severity-threshold: 3

      - name: Upload SARIF
        uses: github/codeql-action/upload-sarif@v3
        with:
          sarif_file: results.sarif
```

## Inputs

| Name                 | Required | Default                                                | Description                                                                                   |
| -------------------- | -------- | ------------------------------------------------------ | --------------------------------------------------------------------------------------------- |
| `code-analyzer`      | no       | `true`                                                 | Run the Salesforce Code Analyzer.                                                              |
| `prettier`           | no       | `false`                                                | Run a Prettier formatting check.                                                              |
| `eslint`             | no       | `false`                                                | Run ESLint and produce a report (e.g. for Sonar).                                             |
| `workspace`          | no       |                                                        | Comma-separated files/folders for the Code Analyzer. Defaults to the current directory.       |
| `rule-selector`      | no       | `Recommended`                                          | Comma-separated Code Analyzer rule selectors, e.g. `Security,Performance`.                     |
| `config-file`        | no       |                                                        | Path to a Code Analyzer configuration file (`code-analyzer.yml`).                             |
| `output-file`        | no       |                                                        | Comma-separated Code Analyzer output files; format derived from the extension (`.sarif`, `.html`, `.json`, …). |
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

## References

- [Salesforce Code Analyzer](https://developer.salesforce.com/docs/platform/salesforce-code-analyzer/overview) | Documentation
- [code-analyzer run](https://developer.salesforce.com/docs/platform/salesforce-code-analyzer/references/code-analyzer-command-reference) | Command Reference
- [Prettier](https://prettier.io/docs/en/cli.html) | CLI
- [ESLint](https://eslint.org/docs/latest/use/command-line-interface) | CLI

## Releases

Latest release notes can be found on the [release page](https://github.com/svierk/sfdx-code-review/releases).

## License

The scripts and documentation in this project are released under the [MIT License](https://github.com/svierk/sfdx-code-review/blob/main/LICENSE).
