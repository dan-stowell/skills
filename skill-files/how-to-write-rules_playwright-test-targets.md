---
name: how-to-write-rules_playwright-test-targets
description: Instructs how to author Bazel targets that run Playwright tests using browsers provided by the rules_playwright toolchain. Use when adding or updating Playwright test runners in Bazel workspaces that consume this repo.
---

## Goal

Author Bazel targets that invoke Playwright's test runner while sourcing browser binaries from `rules_playwright`. Follow the steps below whenever you add a new test target or migrate an existing one.

## 1. Configure the Playwright repository (MODULE.bazel)

```python
bazel_dep(name = "rules_playwright", version = "<match lockfile>")

playwright = use_extension("@rules_playwright//playwright:extensions.bzl", "playwright")
playwright.repo(
    name = "playwright",  # Add a suffix if you need multiple versions
    playwright_version = "1.50.1",  # Keep in sync with package.json/pnpm-lock.yaml
    # integrity_path_map = { "builds/...zip": "sha256-..." },  # Optional: pin downloads
)
use_repo(playwright, "playwright")
```

- Match `playwright_version` to the version in your `package.json` to avoid runtime mismatches.
- When builds are locked down, generate integrity pins with `playwright_integrity_map` (see step 4) and populate `integrity_path_map`.

## 2. Link npm dependencies

In each package that owns Playwright tests, expose the `@playwright/test` runtime:

```python
load("@npm//:@playwright/test/package_json.bzl", playwright_bin = "bin")
load("@npm//:defs.bzl", "npm_link_all_packages")

npm_link_all_packages(name = "node_modules")
```

- `npm_link_all_packages` makes `//:node_modules/@playwright/test` available for `data`.

## 3. Declare the test target

Start with the smallest working target and expand as needed.

```python
playwright_bin.playwright_test(
    name = "e2e",
    args = ["test"],  # Forwarded to `npx playwright test`
    data = [
        "playwright.config.ts",
        "//:node_modules/@playwright/test",
        "//tests:all_specs",  # bundle the spec files
        "@playwright//:chromium-headless-shell",  # Browser payload
    ],
    env = {
        "PLAYWRIGHT_BROWSERS_PATH": "$(rootpath @playwright//:chromium-headless-shell)/../",
    },
    tags = [
        # Add "no-sandbox" on macOS when Firefox is selected.
        # Add "requires-network" if tests need live HTTP access.
    ],
)
```

Key points:

- Every browser you intend to run must be listed in `data` so Bazel ships it to the test sandbox.
- `PLAYWRIGHT_BROWSERS_PATH` must point one directory above a browser archive; use `$(rootpath ...)` so the test works remotely.
- Group test files into a `filegroup` (e.g., `glob(["tests/**/*.spec.ts"])`) to keep `data` readable and hermetic.

### Run multiple browsers

```python
ALL_BROWSERS = [
    "@playwright//:chromium-headless-shell",
    "@playwright//:firefox",
    "@playwright//:webkit",
]

playwright_bin.playwright_test(
    ...
    data = DATA + ALL_BROWSERS,
    env = ENV,
    tags = ["no-sandbox"],  # needed when Firefox runs on macOS
)
```

- Add a boolean like `CI` or `UPDATE_SNAPSHOTS` to `env` when the config switches behaviour.
- Use `select()` if different platforms should download different browsers.

### Alternative entrypoints

- `playwright_bin.playwright_binary` behaves like `npx playwright`; reuse it for UI exploration (`--ui`), snapshot updates, or smoke checks.
- Always reuse the same `DATA` and `ENV` dictionaries so every tool has the same files and browser path.

## 4. Pin browser downloads (optional)

Lock the exact archives `rules_playwright` may download:

```python
load("@rules_playwright//playwright:defs.bzl", "playwright_browser_matrix", "playwright_integrity_map")

playwright_integrity_map(
    name = "playwright_integrity_map",
    browsers = playwright_browser_matrix(
        browser_names = ["chromium-headless-shell", "firefox", "webkit"],
        platforms = ["mac14-arm64", "ubuntu20.04-x64"],
        playright_repo_name = "playwright",
    ),
)
```

1. Run `bazel build //path:playwright_integrity_map`.
2. Copy the emitted `integrity_path_map` into `playwright.repo`.
3. Keep the matrix in sync with the browsers you declare in test `data`.

## 5. Validate targets

1. `bazel test //path:e2e` – verify the target runs with a single browser.
2. Repeat with `--config` or `--test_tag_filters` that select each supported browser.
3. On macOS with Firefox, ensure `no-sandbox` is present or the test will fail at startup.
4. Set `--@rules_playwright//:macos_version` / `--@rules_playwright//:linux_distro` when testing platform-specific downloads.

## Cheatsheet

- Always ship browsers via `data` and point `PLAYWRIGHT_BROWSERS_PATH` one directory up.
- Share `DATA` and `ENV` dictionaries across targets to avoid accidental drifts.
- Generate integrity pins whenever CI must stay deterministic or when network access is restricted.
- Prefer `playwright_binary` targets for local iteration tasks (`--ui`, `--update-snapshots`) so they are hermetic and share the same toolchain setup.
