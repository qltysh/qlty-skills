---
name: setup-coverage
description: Set up and troubleshoot Qlty code coverage reporting in CI pipelines. Use this skill when users want to add coverage uploads to CI (GitHub Actions, CircleCI), integrate Qlty coverage into a project, configure coverage merging or coverage tags for monorepos, fix coverage upload issues (format errors, 0% coverage, path validation failures), or use qltysh/qlty-action/coverage. Supports Python, JavaScript, TypeScript, Go, Ruby, Java, Kotlin, PHP, Elixir, .NET, and Swift. Do NOT use for Qlty static analysis (qlty check, qlty.toml), writing tests, or general CI/CD changes unrelated to coverage.
---

# Set Up Qlty Code Coverage

You are a senior software engineer setting up code coverage reporting in CI for this project using Qlty Cloud. Work through these three phases sequentially.

---

## Phase 1: Analyze Test Suites and Existing Coverage

Analyze the CI configuration and test setup. For each test suite running in CI, document:

```
## Test Suite: [NAME]

Test command: [COMMAND]
Programming language: [LANGUAGE]
Test runner: [TOOL]
Location: [FILE] line [LINE]
Using parallelism: [YES/NO]
Generating code coverage data: [YES/NO]
Coverage output file: [PATH or N/A]
```

Pay close attention to whether coverage data is **already being generated**. Look for:
- Coverage flags in test commands (`--coverage`, `--cov`, `-coverprofile`, etc.)
- Coverage tools configured in build files (JaCoCo in pom.xml/build.gradle, SimpleCov in Gemfile, pytest-cov in requirements.txt, etc.)
- Coverage report files being produced (coverage.xml, lcov.info, .resultset.json, coverage.out, etc.)
- Existing coverage uploads to other services (Codecov, Coveralls, etc.)

### Rules

- Ignore any test suites which are not running in continuous integration.
- Ignore test suites commented out in the continuous integration workflow files.
- If the repo has no tests, or no source code to instrument (e.g., it's a distribution/wrapper repo where the actual code lives elsewhere), **stop here** and report that code coverage setup is not applicable for this repository. Explain why and suggest where coverage should be set up instead.

Do NOT make any changes yet. Just produce the report and move on to Phase 2.

---

## Phase 2: Plan Coverage Uploads

Fetch and read the following Qlty documentation pages to inform your decisions:

- https://docs.qlty.sh/coverage/generating-data
- https://docs.qlty.sh/coverage/quickstart
- https://docs.qlty.sh/coverage/ci
- https://docs.qlty.sh/coverage/path-fixing
- https://docs.qlty.sh/coverage/merging
- https://docs.qlty.sh/coverage/tags

IMPORTANT: Only access URLs from the https://docs.qlty.sh domain.

### Key Principle: Prefer existing coverage data

If the project is already generating coverage data, your plan should just add the Qlty upload step pointing at the existing coverage files. Do NOT reconfigure how coverage is generated unless the existing format is incompatible with Qlty. Avoid adding new dependencies or changing test commands when coverage output already exists.

If the project is NOT generating coverage data yet, then add the minimal configuration needed to produce a coverage report in a format Qlty supports (see the Qlty docs for supported formats per language).

### Decisions to Make

- **Authentication method** (in order of preference):
  1. **OIDC** (best option when supported — no secrets to manage)
  2. **Workspace-level coverage token** (covers all projects in the workspace — easier to manage). Found in the Qlty UI at: `https://qlty.sh/gh/<org>/settings/coverage`
  3. **Project-level coverage token** (scoped to a single project). Found in the Qlty UI at: `https://qlty.sh/gh/<org>/projects/<repo>/settings/coverage/reports`
- **Which test suites to include?**
  - Include all unit test suites (most important for line coverage)
  - Include integration test suites only if they provide significant additional coverage value
  - Include end-to-end (E2E) test suites if they generate code coverage data
- **Coverage merging or tags?**
  - If only one test suite: no merging or tags needed
  - If multiple test suites: choose between server-side coverage merging and coverage tags:
    - **Coverage tags**: Best for monorepos with independent services/packages where you want per-component coverage visibility and carry-forward when only some components' tests run. Use `tag: <name>` on each upload — no `coverage-complete` job needed.
    - **Server-side merging**: Best when multiple test suites cover the same codebase and you want a single combined coverage number. Two approaches:
      - **Parts count** (`total-parts-count: N`): Use when the number of uploads is fixed and known (e.g., a matrix with a static list). Simpler — no separate completion job needed.
      - **Incomplete/complete**: Use when the number of uploads is dynamic or unknown. Each upload uses `incomplete: true`, then a final job signals completion with `command: complete`. Example:
        ```yaml
        # In each test job:
        - uses: qltysh/qlty-action/coverage@v2
          with:
            files: coverage.xml
            incomplete: true
        # In the final job (needs all test jobs):
        - uses: qltysh/qlty-action/coverage@v2
          with:
            command: complete
        ```
        IMPORTANT: Use `command: complete`, NOT `complete: true` — the latter is not a valid input and will silently fail.
  - If tests run with concurrency (e.g., pytest-xdist): coverage merging happens client-side within the test runner, so no server-side merging is needed
- **Upload method**: Qlty GitHub Action (`qltysh/qlty-action/coverage`) or manual CLI install?
  - CRITICAL: On GitHub Actions, ALWAYS use the official Qlty GitHub Action. Do NOT install the Qlty CLI manually.
  - On CircleCI, use the Qlty CircleCI orb (`qltysh/qlty-orb`). Read the orb documentation at https://circleci.com/developer/orbs/orb/qltysh/qlty-orb for usage details. If the customer is unable to use the orb, fall back to installing the Qlty CLI manually.

Produce a concise plan document with your decisions, then move on to Phase 3.

---

## Phase 3: Implement

Execute the plan:

### 1. Modify CI Configuration

- If coverage is already being generated: just add the Qlty upload step after the existing test/coverage step. Point `files:` at the existing coverage output file. Do not change the test command or coverage tooling.
- If coverage is NOT being generated: add the minimal changes needed to produce coverage output, then add the upload step.
- Configure authentication (OIDC or token) as specified
- Implement coverage merging or tags as specified
- Ensure minimal changes to existing workflows

### 2. Create Branch and Pull Request

- Create a feature branch (e.g., `qlty-coverage-integration`)
- Commit changes with a clear message
- Push to remote
- Open a PR with:
  - Clear title and description
  - Summary of changes
  - Manual steps needed (e.g., adding secrets)
  - Link to PR in your output

### 3. Monitor and Debug the Build

- Wait for CI to start
- Monitor only the **test workflow run** you modified — ignore unrelated PR checks (e.g., static analysis, formatting, linting) that are not part of the coverage setup
- If the test workflow fails:
  - Diagnose the error
  - Fix the issue
  - Commit and push
  - Repeat until success or unable to diagnose
- After the workflow passes, check the **Qlty coverage upload step logs** to verify coverage data was actually published. Look for confirmation like "Line Coverage: X%" or "Published" in the output. A green checkmark alone is not sufficient — the upload step may silently skip errors.
- If the upload shows **0% coverage** (0 covered lines but many uncovered lines), the coverage report format may be wrong. For Android/Kotlin projects, try using the JaCoCo Gradle plugin explicitly instead of `createDebugUnitTestCoverageReport`, and set `format: jacoco` on the upload step.
- If the upload shows **parse errors** (`missing field 'packages'` or similar), Qlty likely misdetected the coverage format. Fix by adding an explicit `format:` parameter on the upload step (e.g., `format: clover` for PHP Clover XML, `format: jacoco` for JaCoCo XML). This is a known issue with PHP Clover files being misdetected as Cobertura.
- If the upload shows **path validation errors** (files missing on disk), fix with `add-prefix` or `strip-prefix` options. For Java/Kotlin multi-module projects, set the `JACOCO_SOURCE_PATH` environment variable with all module source paths. Read https://docs.qlty.sh/coverage/path-fixing for details.
- Report final status

### Constraints

- NEVER create files unless absolutely necessary — prefer editing existing CI files
- NEVER add new test suites
- NEVER set `validate: false` for path validation
- NEVER add secrets yourself — provide instructions only
- ONLY modify files required for Qlty coverage integration
- CRITICAL: On GitHub Actions, ALWAYS use the official Qlty GitHub Action (`qltysh/qlty-action/coverage`) for uploads. Do NOT install the Qlty CLI manually.
- Do NOT change existing coverage generation tooling unless the format is incompatible with Qlty

### Final Report

When done, report:
1. Files modified/created
2. PR URL
3. Build status (success or issues encountered)
4. Manual steps required (if any)
5. Link to verify the coverage report in Qlty: `https://qlty.sh/gh/<org>/projects/<repo>/settings/coverage/reports`
6. Next steps for the user:
   - Confirm the uploaded report appears on the coverage reports page above
   - After merging the PR to the default branch, verify coverage shows on the project's Overview page
   - Optionally, enable coverage commit status checks in the project's Review Config settings (`https://qlty.sh/gh/<org>/projects/<repo>/settings/review`) to enforce coverage thresholds on PRs
