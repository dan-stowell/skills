---
name: authoring-rules_cypress-tests
description: Guides Bazel authors through creating Cypress test targets with rules_cypress; use when defining cypress_test or cypress_module_test in this repo or dependent workspaces.
---

## Overview
- Prefer `cypress_test` when the Cypress CLI (`cypress run`, `cypress open`, etc.) covers the workflow.
- Use `cypress_module_test` only when the test runner must call the Cypress module API directly (custom retries, result post-processing, orchestration hooks).
- Both rules extend `js_test`, so all common attributes (`data`, `args`, `env`, `chdir`, `timeout`, `tags`, …) behave as in `rules_js`.

## Quick Start
1. Register the Cypress toolchain once per workspace; omit `cypress_integrity` when you consume a version already mirrored in this repo, and supply hashes only for custom mirrors:
  ```starlark
  # WORKSPACE.bazel
  load("@aspect_rules_cypress//cypress:dependencies.bzl", "rules_cypress_dependencies")
  rules_cypress_dependencies()

  load("@aspect_rules_cypress//cypress:repositories.bzl", "cypress_register_toolchains")
  cypress_register_toolchains(
      name = "cypress",
      cypress_version = "13.13.0",          # pick a version
  )
  ```
  ```starlark
  # MODULE.bazel
  bazel_dep(name = "aspect_rules_cypress", version = "<release>")
  cypress = use_extension("@aspect_rules_cypress//cypress:extensions.bzl", "cypress")
  cypress.toolchain(cypress_version = "13.13.0")
  use_repo(cypress, "cypress_toolchains")
  register_toolchains("@cypress_toolchains//:all")
  ```
2. Expose a linked `node_modules` tree so the default `cypress = "//:node_modules/cypress"` label resolves:
  ```starlark
  load("@npm//:defs.bzl", "npm_link_all_packages")
  npm_link_all_packages(name = "node_modules")
  ```
3. Define tests; see `e2e/workspace/cli_test/BUILD` for a CLI target and `e2e/workspace/module_test/BUILD.bazel` for a module-runner example you can copy.

## Prerequisites
- Toolchains: Register once using `cypress_register_toolchains` (WORKSPACE) or the `cypress` module extension (bzlmod). Omit `cypress_integrity` when you consume a version mirrored in this repo; supply per-platform hashes only for custom mirrors.
- Node modules: Provide `//:node_modules` via `npm_link_all_packages`. When using `npm_translate_lock`, set `lifecycle_hooks_exclude = ["cypress"]` so npm does not download an extra Cypress binary that conflicts with the Bazel-managed one.
- Data dependencies: Make every config file, spec, fixture, and helper binary addressable as a Bazel target and add it through `data` or other attributes the rule propagates.

## Canonical CLI Target (`cypress_test`)
```starlark
load("@aspect_rules_cypress//cypress:defs.bzl", "cypress_test")

cypress_test(
    name = "cli_test",
    args = [
        "run",
        "--config-file=cli_test/cypress.config.ts",
        # "--spec=cli_test/cli_test.cy.ts",  # uncomment when you only want this spec
    ],
    data = [
        "cypress.config.ts",
        "cli_test.cy.ts",
        "tsconfig.json",
        "//:node_modules",
        # "//app/server:dev_server",  # add helpers your tests require
    ],
    timeout = "short",
    # chdir = package_name(),        # enable package-relative paths when configs expect them
)
```
- `args` go directly to the Cypress CLI. Paths resolve from the Bazel runfiles root unless you set `chdir = package_name()` to mirror the package-relative layout developers use locally. Uncomment `--spec` only when you need to narrow the run to specific specs.
- List every file or binary Cypress reads in `data` (`cypress.config.*`, spec files, fixtures, support directories, helper servers). Prefer explicit lists over `glob` for reproducibility.
- `disable_sandbox` defaults to `True`, adding the `no-sandbox` tag; only flip it off after verifying Cypress works inside the sandbox on all platforms.
- Use Bazel tags to communicate CI behaviour (for example `["manual"]` for `cypress open` targets or `["requires-network"]` when tests hit live services). See `e2e/workspace/cli_test/BUILD` for a minimal working target.

## Module API Target (`cypress_module_test`)
```starlark
load("@aspect_rules_cypress//cypress:defs.bzl", "cypress_module_test")

cypress_module_test(
    name = "module_test",
    runner = "runner.js",
    data = [
        "runner.js",
        "cypress.config.js",
        "module_test.cy.js",
        "//:node_modules",
    ],
    # chdir = package_name(),
)
```
- `runner` is required and is executed with `CYPRESS_RUN_BINARY` already set. A minimal implementation:
  ```javascript
  // runner.js
  const cypress = require("cypress");

  async function main() {
    const result = await cypress.run({ headless: true });
    if (result.status !== "finished") {
      console.error("Cypress exited with status:", result.status);
      process.exit(2);
    }
    if (result.failures) {
      process.exit(1);
    }
    process.exit(0);
  }

  main().catch((err) => {
    console.error("Unexpected error running Cypress", err);
    process.exit(3);
  });
  ```
- Place the runner, config, specs, and any helpers in `data`, and set `chdir` only if your runner assumes package-relative paths.
- Use this rule when you must inspect `result.runs`, rerun failing specs, or publish artifacts mid-build. See `e2e/workspace/module_test/BUILD.bazel` for a complete example.

## Managing Browsers and Extra Tools
- Add browsers through the `browsers` attribute, pointing to archive or directory labels (for example `@chrome_linux//:all`). Bazel places them at the runfiles root, so reference them from Cypress with paths like `../../../chrome_linux`.
- Keep large browser archives out of `data` to avoid extra copies; fetch them via external repositories as shown in this repo’s `MODULE.bazel`.

## Running and Debugging
- Execute tests with `bazel test //path:cli_test` (`--test_output=all` surfaces Cypress logs). Reserve `bazel run` for interactive flows such as `cypress open`.
- If Cypress cannot locate config or spec files, inspect runfiles with `bazel cquery --output=starlark` or `bazel test --run_under='ls -R .'` to confirm `data` coverage.
- For flaky UI flows, set `env = {"CYPRESS_RETRIES": "2"}` or pass `--record`/`--config` via `args`. Keep environment overrides in the target definition for reproducibility.

## Common Gotchas Checklist
- [ ] `//:node_modules` provides the Cypress package version your tests expect.
- [ ] Toolchains are registered for every CI platform before `bazel test` runs.
- [ ] Path expectations in `cypress.config.*` match your `chdir` setting (unset ⇒ runfiles root, set ⇒ package-relative).
- [ ] CI tags (`no-sandbox`, `requires-network`, `exclusive`, …) on each target match how the tests execute.
