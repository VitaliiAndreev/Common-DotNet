# Common-DotNet

Shared .NET CI building blocks for SynergyOps repositories. Mirrors the
role `Common-PowerShell` plays for PowerShell modules: a single
source of truth for reusable GitHub Actions workflows (and, later,
MSBuild props, analyzer rulesets, base test SDKs) that every SynergyOps
.NET repo consumes by `uses:` reference pinned to a commit SHA.

## Index
- [Status](#status)
- [Workflows](#workflows)
  - [ci-dotnet.yml](#ci-dotnetyml)
- [Artifacts](#artifacts)
- [Composite actions](#composite-actions)
- [Self-test sample](#self-test-sample)
- [Linting and local checks](#linting-and-local-checks)
  - [Running the checks locally](#running-the-checks-locally)
- [Branch protection](#branch-protection)
- [Implementation docs](#implementation-docs)

## Status
Bootstrap phase. The reusable `ci-dotnet.yml` workflow is being built up
step-by-step per the plan under `docs/dev/implementation/`. No tagged
release exists yet; consumers should not pin to this repo until the
first release lands.

## Workflows

### ci-dotnet.yml
Reusable workflow that performs the .NET CI preflight, restore, build,
test, code coverage collection, human-readable coverage reporting (with
the report uploaded as a build artifact), and a coverage threshold gate
that fails the job when line coverage drops below a consumer-configurable
bar.

It has three `on:` triggers - `workflow_call` for consumers,
`pull_request` (targeting `master`) so it **self-triggers in this repo**
(validating itself against the [self-test sample](#self-test-sample) with
no separate caller file), and `workflow_dispatch` so a maintainer can run
it manually with a one-off `runner` override. This is the same one-file
pattern the other SynergyOps reusable-workflow repos use (e.g.
Common-Automation's `ci-bash.yml`). There is no `push` trigger - a PR to
`master` already runs this gate before merge, so a post-merge re-run
would be redundant. A workflow's `on:` triggers fire only in the repo
that hosts the file, so the self-trigger is invisible to consumers, who
reach it only via `uses:`. Because input defaults do not apply on
`pull_request` (the `inputs` context is populated only for
`workflow_call` and `workflow_dispatch`), every input is empty-safe:
`solution-path` auto-discovers, `dotnet-version` is coalesced to its
default at the use site, and `coverage-threshold` is defaulted inside its
composite.

**Inputs:**
- `runner` (string, optional, default empty) - one-off `runs-on`
  override that takes precedence over the `CI_DOTNET_RUNNER` variable.
  A JSON label array (e.g. `["ubuntu-latest"]`), same format as the
  variable. Settable via `workflow_dispatch` (manual run) or a
  consumer's `with:`; not settable on this repo's automatic
  `pull_request` runs. Leave empty to use the variable.
- `solution-path` (string, optional, default empty) - path to the
  `.sln`/`.slnx` to restore and build, relative to the consumer repo
  root. When omitted, the
  [`resolve-solution-path`](.github/actions/resolve-solution-path/) step
  auto-discovers a single solution file and fails if there are zero or
  more than one. Pass it explicitly to disambiguate a multi-solution
  repo.
- `dotnet-version` (string, optional, default `10.0.x`) - .NET SDK
  version installed when running on a GitHub-hosted runner, in
  `actions/setup-dotnet` syntax. Unused on self-hosted.
- `coverage-threshold` (number, optional, default `90`) - minimum
  acceptable line coverage as a percentage (0-100). The threshold
  gate step fails the job when the merged Cobertura report's
  `line-rate` falls below this value. Values above 100 are silently
  capped at 100; negative values are rejected.

**Variables:**
- `CI_DOTNET_RUNNER` (repository or organization variable, optional) -
  the `runs-on` selector consumers use to pick the runner without
  touching their caller YAML. A JSON label array - `["ubuntu-latest"]`
  for a single label, or `["self-hosted","linux","gpu"]` for multi-label
  targeting. Unset selects any self-hosted runner. See the
  **Selecting the runner** note below.

**Orchestration model:** the workflow YAML is an orchestrator. Each
step delegates to a dedicated composite action under
`.github/actions/`, so per-step logic stays testable in isolation and
the workflow stays readable. Composite actions colocate their
PowerShell script next to `action.yml` and invoke it via
`${{ github.action_path }}`, which means action body and script ship
together at the same SHA. The workflow runs a second
`actions/checkout` of Common-DotNet into `.common-dotnet/` and
references each composite action as `./.common-dotnet/.github/actions/<name>`,
so action resolution is always against a known local path rather
than a remote ref - see [Why two checkouts](#why-two-checkouts).

**Job steps (in order):**
1. `actions/checkout@v5` - check out the consumer repo (the code
   being restored, built, tested, and measured).
2. `actions/checkout@v5` - second checkout of `Common-DotNet` itself
   into `.common-dotnet/`, used only as the source of the composite
   actions every later step references. The `ref:` is
   `github.workflow_sha` - the commit SHA of the called workflow
   file - so a consumer that pinned `uses: ...ci-dotnet.yml@<sha>`
   gets the composite actions from exactly that SHA; no master leak
   across the SHA-pinned boundary. See
   [Why two checkouts](#why-two-checkouts) below.
3. [`clear-workspace-artifacts`](.github/actions/clear-workspace-artifacts/) -
   removes `bin/`, `obj/`, `TestResults/`, `CoverageReport/` anywhere
   under the workspace. Required because self-hosted runners persist
   `_work/` across jobs and stale output from a prior run can poison
   the build.
4. [`resolve-solution-path`](.github/actions/resolve-solution-path/) -
   resolves the solution to build and publishes it as `$SOLUTION_PATH`
   for every step below. Uses an explicit `solution-path` input when
   given, else auto-discovers a single `.sln`/`.slnx` and fails on zero
   or multiple matches. Runs after cleanup so a stale solution under
   `bin/obj` cannot be discovered.
5. [`provision-dotnet-toolchain`](.github/actions/provision-dotnet-toolchain/)
   - installs the .NET SDK (via `actions/setup-dotnet`) and the
   ReportGenerator global tool, which a GitHub-hosted image does not
   carry. The workflow gates this call on `runner.environment`, so it
   runs **only on GitHub-hosted runners** and is skipped on self-hosted,
   where the runner image is expected to carry the toolchain pre-baked
   (external to this workflow). Gating on the runtime environment rather
   than a runner label is
   deliberate - a self-hosted pool is targeted by an arbitrary custom
   label that no inspection could classify. Runs before the asserts so
   on the hosted path they confirm the install rather than failing on a
   bare image.
6. [`assert-dotnet-sdk`](.github/actions/assert-dotnet-sdk/) - runs
   `dotnet --version` and fails fast with a message pointing at the
   runner image if the SDK is missing, instead of letting `dotnet
   restore` produce a confusing error mid-job.
7. [`assert-reportgenerator`](.github/actions/assert-reportgenerator/) -
   verifies the ReportGenerator global tool is on PATH and fails fast
   with the same runner-image guidance if missing.
   Grouped with the SDK assertion so every required tool is checked
   before any long-running step.
8. [`dotnet-restore`](.github/actions/dotnet-restore/) - `dotnet
   restore <solution-path>`.
9. [`dotnet-build`](.github/actions/dotnet-build/) - `dotnet build
   <solution-path> --no-restore`.
10. [`dotnet-test`](.github/actions/dotnet-test/) - `dotnet test
    <solution-path> --no-build --collect:"XPlat Code Coverage"
    --results-directory ./TestResults`. Separated from build so the
    workflow exposes two distinct failure surfaces (compile errors vs
    test failures), and so a future "build-only" mode can be added by
    gating this step on an input without disturbing the build pipeline.
    Coverage output lands at `TestResults/<guid>/coverage.cobertura.xml`
    for the report and threshold steps that follow.
11. [`dotnet-coverage-report`](.github/actions/dotnet-coverage-report/) -
    runs ReportGenerator over the Cobertura output from the test step
    and writes `CoverageReport/{index.html, Summary.txt, Cobertura.xml}`.
    The merged `Cobertura.xml` is the canonical input the threshold
    gate reads, so the gate does not need to glob `TestResults/`.
12. [`assert-coverage-threshold`](.github/actions/assert-coverage-threshold/) -
    reads `line-rate` from `CoverageReport/Cobertura.xml` and fails
    the job with a diagnostic naming both the observed value and the
    configured threshold when coverage drops below `coverage-threshold`.
    Placed before the artifact upload so the gate is the final word on
    the build; the upload step's `if: always()` guarantees reviewers
    still get the report when the gate fails - that is exactly when
    they need it most.
13. `actions/upload-artifact@v4` - uploads the entire `CoverageReport/`
    directory as a single artifact named `coverage-report`. Runs with
    `if: always()` so reviewers can still inspect partial coverage if
    the test step or the threshold gate failed.

Cleanup runs **before** the SDK assertion so that a prior run's
artifacts do not survive into a job that aborts at the assertion;
the assertion in turn runs **before** restore so SDK absence is
diagnosed at the right step.

**Why two checkouts:** the alternative (if-guarded pairs of `./` and
`<repo>@master` step references) forces GitHub Actions to resolve
the `@master` ref at job start even when the if-guard would skip the
step, so a stale `master` (or a PR-branch action that does not yet
exist on `master`) breaks the job before any step runs. A separate
checkout of Common-DotNet into `.common-dotnet/`, with every step
referencing `./.common-dotnet/.github/actions/<name>`, side-steps
the eager-resolution footgun entirely. A small side-benefit: when this
repo self-triggers on a PR, the run exercises the PR's own composite
changes end-to-end, because the self path uses `./` (the PR head
workspace) rather than a pinned `master` ref.

**Selecting the runner:** `runs-on` is resolved by precedence - the
`runner` input override (highest), then the `CI_DOTNET_RUNNER` repository
or organization variable, then any self-hosted runner. Steady-state
selection lives in the variable; the input is for one-off overrides
(`workflow_dispatch` or a consumer's `with:`). The variable accepts
three forms (the input accepts the same):

| `CI_DOTNET_RUNNER` | `runs-on` | use |
| --- | --- | --- |
| *(unset)* | `[self-hosted]` | any self-hosted runner |
| `["ubuntu-latest"]` | `[ubuntu-latest]` | public repo / GitHub-hosted |
| `["dotnet-build"]` | `[dotnet-build]` | one self-hosted pool, by unique label |
| `["self-hosted","linux","gpu"]` | `[self-hosted, linux, gpu]` | a specific runner, by label intersection |

The value is parsed as a JSON label array (the job runs on a runner
carrying *all* listed labels). One consistent format - even a single
label is bracketed, e.g. `["ubuntu-latest"]`. A non-JSON value fails
`fromJSON` at job setup, so a typo surfaces immediately.

The intended consumer model maps to the GitHub billing split - private
repos meter Actions minutes, public repos do not:
- **Private repos:** leave `CI_DOTNET_RUNNER` unset (any self-hosted) or
  set it to a pool label / JSON array to pin a specific runner. Minutes
  are not metered.
- **Public repos:** set `CI_DOTNET_RUNNER` to a hosted label (e.g.
  `ubuntu-latest`). No caller-YAML change.

A configuration variable (not an input) is used for *steady-state*
selection so the choice lives in settings and a thin caller workflow
needs no runner argument at all. An org-level `CI_DOTNET_RUNNER` with
per-repo overrides lets the whole org default one way and exceptions opt
out. The `runner` input sits above it only for one-off runs - e.g.
manually dispatching against a hosted runner to reproduce a consumer's
environment, without touching the variable.

**Runner requirements:** every composite action in this workflow declares
`shell: pwsh`, so **PowerShell (`pwsh`) is a prerequisite of the workflow
itself on every runner**, hosted or not - it is not part of the .NET
toolchain the `runner.environment` split below governs. GitHub-hosted
images ship it. A self-hosted image must bake it in; without it the job
dies at the first composite that runs a shell with a bare
`pwsh: command not found`, and no `assert-*` preflight below can report
anything better because those are themselves PowerShell.

That is why the job's first step is `assert-pwsh`, a bash-shelled check
that exists purely to turn that failure into a sentence. It is **not** one
of this repo's composites: it lives in
[Common-PowerShell](https://github.com/Klark-Morrigan/Common-PowerShell/tree/master/.github/actions/assert-pwsh),
because the prerequisite belongs to PowerShell's domain rather than
.NET's, and every workflow in this family that writes its steps in `pwsh`
needs the identical check. This workflow stages that repo into
`.powershell-common/` - unconditionally, since Common-DotNet keeps no copy
- and calls it from there.

The .NET toolchain proper comes from a different place depending on where
the job lands, decided at runtime by `runner.environment` (which GitHub
reports as `github-hosted` or `self-hosted`), not by the runner label:
- **Self-hosted:** the .NET SDK *and* the ReportGenerator global tool
  (`dotnet tool install -g dotnet-reportgenerator-globaltool`) are
  expected to be pre-baked into the runner image (how the pool is
  provisioned is the runner operator's concern, external to this
  workflow). Consumers do not install either ad hoc, and the workflow
  skips the
  [`provision-dotnet-toolchain`](.github/actions/provision-dotnet-toolchain/)
  call.
- **GitHub-hosted:** the hosted image carries neither at the required
  version, so `provision-dotnet-toolchain` installs both inline before
  the assert preflights. This is the one sanctioned exception to
  "consumers do not install ad hoc": the install is centralized in a
  composite action that the workflow gates on `runner.environment`, so
  the single-source-of-truth for self-hosted provisioning is left
  intact. Gating on the environment rather than the label is what makes
  a self-hosted *pool* (an arbitrary custom label) correctly skip the
  install.

**Coverage contract:** the test step activates Coverlet's
`XPlat Code Coverage` data collector, which requires each consumer
test project to reference the `coverlet.collector` NuGet package
(currently `6.0.4` in the self-test sample). Without that
PackageReference, `dotnet test` still passes but no
`coverage.cobertura.xml` is produced and the downstream report and
threshold steps fail. Output path is
`TestResults/<guid>/coverage.cobertura.xml`, relative to the working
directory the test step runs in.

**Self-trigger coverage:** when this repo self-triggers (see the `on:`
note above), the run mirrors real consumer behavior - it lands on
whatever `CI_DOTNET_RUNNER` (or the self-hosted default) selects and
omits `solution-path`, so it exercises auto-discovery against the
[self-test sample](#self-test-sample). Only the configured path runs
each time, so path-specific logic on the other side (e.g. the
hosted-only toolchain provisioning) is not covered until you flip
`CI_DOTNET_RUNNER` on this repo. That is a deliberate trade-off: one
faithful consumer-mirror run rather than both paths on every PR.

## Artifacts
The workflow uploads one artifact per job run:

- `coverage-report` - the entire `CoverageReport/` directory, which
  contains:
  - `index.html` - browsable HTML report.
  - `Summary.txt` - plain-text summary; includes the `Line coverage:`
    line the threshold gate (Step 12) parses.
  - `Cobertura.xml` - merged Cobertura across all input report files;
    the canonical input for the threshold gate.
  - Supporting CSS/JS/SVG/per-class HTML pages for the HTML report.

The upload step runs with `if: always()`, so the artifact is produced
even when the test step fails - reviewers can still inspect partial
coverage for the assemblies whose tests did run.

## Composite actions
The reusable workflow above is the recommended entry point, but each
composite action is also directly consumable for repos that want
finer control or atomic per-action SHA pinning:

- [`clear-workspace-artifacts`](.github/actions/clear-workspace-artifacts/)
- [`resolve-solution-path`](.github/actions/resolve-solution-path/) -
  optional input: `solution-path`; publishes the resolved solution to
  `$SOLUTION_PATH`, auto-discovering a single `.sln`/`.slnx` when the
  input is empty
- [`provision-dotnet-toolchain`](.github/actions/provision-dotnet-toolchain/) -
  optional input: `dotnet-version` (default `10.0.x`); installs the
  SDK and ReportGenerator unconditionally when called (the caller gates
  on `runner.environment` to run it only on GitHub-hosted runners)
- [`assert-dotnet-sdk`](.github/actions/assert-dotnet-sdk/)
- [`assert-reportgenerator`](.github/actions/assert-reportgenerator/)
- [`dotnet-restore`](.github/actions/dotnet-restore/) - input:
  `solution-path`
- [`dotnet-build`](.github/actions/dotnet-build/) - input:
  `solution-path`
- [`dotnet-test`](.github/actions/dotnet-test/) - input:
  `solution-path`; collects Coverlet coverage to
  `TestResults/<guid>/coverage.cobertura.xml`
- [`dotnet-coverage-report`](.github/actions/dotnet-coverage-report/) -
  optional inputs: `reports-glob` (default
  `TestResults/**/coverage.cobertura.xml`), `target-dir` (default
  `CoverageReport`)
- [`assert-coverage-threshold`](.github/actions/assert-coverage-threshold/) -
  optional inputs: `coverage-threshold` (default `90`),
  `cobertura-path` (default `CoverageReport/Cobertura.xml`); fails
  the step when observed line coverage is below the threshold

## Self-test sample
`tests/sample/` is a minimal .NET solution (one class library plus one
xUnit test project, targeting `net10.0`) whose only job is to give the
in-progress reusable workflow something to build, test, and measure
coverage against. It is scaffolding: once this repo gains real shared
.NET code (MSBuild props, analyzers, a base test SDK, or similar),
that code becomes the natural self-test target and the sample is
retired in favour of it.

Local validation:
```
dotnet build tests/sample/Sample.sln
dotnet test  tests/sample/Sample.sln
```

## Linting and local checks

Two delegating workflows lint this repo's YAML and bash surfaces on every
pull request to `master`. Both forward to reusable workflows in
`Common-Automation`, so the lint logic lives in one place and this repo
carries only thin caller files. They sit alongside `ci-dotnet.yml`, which
remains the .NET build/test/coverage gate:

- [`.github/workflows/ci-yaml.yml`](.github/workflows/ci-yaml.yml) -
  delegates to Common-Automation's reusable `ci-yaml.yml`, which runs
  actionlint, action-validator, yamllint, and ansible-lint in parallel.
  Each job auto-skips when its target surface is absent, so this repo's
  workflows and composite-action YAML get linted at no extra cost.
- [`.github/workflows/ci-bash.yml`](.github/workflows/ci-bash.yml) -
  delegates to Common-Automation's reusable `ci-bash.yml`, which runs
  shellcheck, the `check-sh-executable` +x-bit gate, and every `*.bats`
  suite. This repo's only bash surface is the runner shims under
  `scripts/`, held to the same strict bar as every other repo.

These lint the YAML and bash surfaces only. The .NET sample tests are
unaffected - they run via `dotnet test` (see
[Self-test sample](#self-test-sample)), never through this lint tooling.

### Running the checks locally

Three sibling runner shims reproduce CI locally via Git Bash plus Docker, so
failures surface before the PR rather than in CI. Each is a thin shim that
points Common-Automation's engine at this repo via
`COMMON_AUTOMATION_TARGET_REPO`, so `Common-Automation` must be a sibling
checkout (`..\Common-Automation`). Every `.sh` has a `.bat` twin that is the
Explorer double-click launcher for the same flow.

- [`scripts/run-ci-yaml-and-bash.sh`](scripts/run-ci-yaml-and-bash.sh) (and its
  [`.bat`](scripts/run-ci-yaml-and-bash.bat)) is the **main** local entry: it runs
  the full lint suite **and** the bats tests, the local equivalent of this repo's
  `ci-yaml.yml` + `ci-bash.yml`.
- [`scripts/run-lint-yaml-and-bash.sh`](scripts/run-lint-yaml-and-bash.sh) (and its
  [`.bat`](scripts/run-lint-yaml-and-bash.bat)) runs a single half - the lint suite
  only (shellcheck, actionlint, action-validator, yamllint, ansible-lint).
- [`scripts/run-tests-bash.sh`](scripts/run-tests-bash.sh) (and its
  [`.bat`](scripts/run-tests-bash.bat)) runs the other half - the bats tests only.

These runners are named for YAML/bash lint and bats, distinct from this repo's
real .NET test path (`dotnet test` over the sample solution) - they never touch
the .NET tests.

Two supporting files keep the bash tooling CI-clean on a Windows checkout:

- [`scripts/fix-permissions.sh`](scripts/fix-permissions.sh) (and its
  [`.bat`](scripts/fix-permissions.bat) launcher) re-stages `+x` on every
  tracked `*.sh` missing it, so the `check-sh-executable` gate stays green
  after authoring a script on Windows (where new files land mode `0644`).
- [`.gitattributes`](.gitattributes) pins `*.sh` to LF and `*.bat` to
  CRLF, so a stray CR on a shebang line cannot break the Linux CI runners.

## Branch protection
The self-triggered [`ci-dotnet.yml`](#ci-dotnetyml) run must be a
required status check on `master`. GitHub does not let workflow files
declare their own branch-protection rules, so this is configured
manually once in the repository settings:

1. Repository **Settings** -> **Branches** -> **Branch protection
   rules** -> add or edit the rule for `master`.
2. Enable **Require status checks to pass before merging**.
3. Under the status-check search box, add the check for the `build`
   job (named **`Build, test, and measure coverage`**) of
   `ci-dotnet.yml`. GitHub renders this as either
   `Build, test, and measure coverage` or
   `ci-dotnet / Build, test, and measure coverage` depending on org
   settings - open a recent run's **Checks** tab and copy the exact
   string shown, then point the rule at it. If the workflow or job
   `name:` changes, the rule must be re-pointed - GitHub does not
   auto-update it.
4. Enable **Require branches to be up to date before merging** so
   stale PRs cannot bypass the gate by merging against an older
   `master`.

Without this manual step the workflow still runs on every PR but a
red result is advisory only - merges are not blocked.

## Implementation docs
- [Shared .NET CI - problem](docs/dev/implementation/2%20-%20shared%20dotnet%20ci/problem.md)
- [Shared .NET CI - plan](docs/dev/implementation/2%20-%20shared%20dotnet%20ci/plan.md)
