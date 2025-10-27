# How to Add a Playwright Test Target

This guide walks you through creating end-to-end Playwright tests for BuildBuddy that spin up a full BuildBuddy server and test the UI.

## Prerequisites

The repository should already have the necessary dependencies installed:
- `@playwright/test` in `package.json`
- `rules_playwright` in `deps/bazel_dep.MODULE.bazel`
- Playwright extension configured in `deps/web.MODULE.bazel`

## File Structure

Create a test directory with the following structure:

```
app/<feature>/tests/
├── BUILD.bazel
└── <test-suite-name>/
    ├── playwright.config.js
    ├── test_environment.js
    ├── start_buildbuddy.js
    └── <test-name>.spec.js
```

## Step 1: Create BUILD.bazel

Create a `BUILD.bazel` file with three genrules and a playwright_test target:

```bazel
load("@npm//:@playwright/test/package_json.bzl", playwright_bin = "bin")

# Copy app bundle JavaScript file
genrule(
    name = "app_bundle_js",
    srcs = ["//app:app_bundle"],
    outs = ["app_bundle.js"],
    cmd = "cp $(location //app:app_bundle)/app.js $@",
)

# Copy app styles
genrule(
    name = "style_css",
    srcs = ["//app:style.css"],
    outs = ["style.css"],
    cmd = "cp $< $@",
)

# Copy app bundle hash for cache busting
genrule(
    name = "sha_sum",
    srcs = ["//app:sha"],
    outs = ["sha.sum"],
    cmd = "cp $< $@",
)

playwright_bin.playwright_test(
    name = "my_playwright_test",
    args = ["test", "--config=app/<feature>/tests/<test-suite-name>/playwright.config.js"],
    data = [
        "<test-suite-name>/<test-name>.spec.js",
        "<test-suite-name>/playwright.config.js",
        "<test-suite-name>/start_buildbuddy.js",
        "<test-suite-name>/test_environment.js",
        ":app_bundle_js",
        ":style_css",
        ":sha_sum",
        "//server/cmd/buildbuddy",
        "//:node_modules",
        "@playwright//:chromium-headless-shell",
    ],
    env = {
        "PLAYWRIGHT_BROWSERS_PATH": "$(rootpath @playwright//:chromium-headless-shell)/../",
    },
    tags = ["no-sandbox"],
    timeout = "moderate",
)
```

The genrules copy the web bundle, CSS, and SHA into the test's runfiles—note the `$(location //app:app_bundle)/app.js` reference because `app_bundle` expands to a directory. The `data` attribute must list every file the runner needs, and keep `tags = ["no-sandbox"]` so Bazel allows the embedded server to start.

## Step 2: Create test_environment.js

This file defines shared configuration:

```javascript
const HTTP_PORT = 19080;

module.exports = {
  HTTP_PORT,
  HEALTH_URL: `http://127.0.0.1:${HTTP_PORT}/healthz?server-type=buildbuddy-server`,
};
```

**Important:** The `?server-type=buildbuddy-server` query parameter is required for the health check to pass. Without it, the server's liveness check will fail.

## Step 3: Create start_buildbuddy.js

This script starts the BuildBuddy server for testing. Copy the reference implementation at `app/invocation/tests/fullstack_empty/start_buildbuddy.js` and tweak it for your suite. The script should:
- Create a temporary workspace with CAS and SQLite paths.
- Write a minimal config pointing at those directories.
- Spawn `//server/cmd/buildbuddy` with `--config_file`, `--port`, `--ssl_port=0`, `--monitoring_port=0`, `--grpc_port=0`, `--app_directory=app`, and `--server_type=buildbuddy-server`.
- Clean up the temporary workspace on exit.

## Step 4: Create playwright.config.js

Configure Playwright to use your server:

```javascript
const { defineConfig } = require("@playwright/test");
const path = require("path");
const { HTTP_PORT, HEALTH_URL } = require("./test_environment");

const startScript = path.join(__dirname, "start_buildbuddy.js");

module.exports = defineConfig({
  testDir: __dirname,
  retries: 0,
  use: {
    baseURL: `http://127.0.0.1:${HTTP_PORT}`,
    headless: true,
    trace: "retain-on-failure",
  },
  webServer: {
    command: `node ${startScript}`,
    url: HEALTH_URL,
    stdout: "pipe",
    stderr: "pipe",
    reuseExistingServer: false,
    timeout: 120 * 1000,
  },
});
```

**Key settings:**
- `webServer.url`: Playwright will wait for this URL to return 200 before running tests
- `reuseExistingServer: false`: Ensures a clean server instance for each test run

## Step 5: Write Your Tests

Author tests using Playwright's standard `test` and `expect` APIs. For inspiration, see `app/invocation/tests/fullstack_empty/empty_state.spec.js`, which verifies the Quickstart page loads and unknown invocations show an error message.

## Running Tests

```bash
# Run all tests in the target
bazel test //app/<feature>/tests/...

# Run with streamed output for debugging
bazel test //app/<feature>/tests/... --test_output=streamed
```

## Troubleshooting

### Port already in use
If tests fail with "port already in use", kill any lingering processes:
```bash
lsof -ti:19080 | xargs kill -9
```

### Health check timing out
Verify the `HEALTH_URL` includes the `?server-type=buildbuddy-server` query parameter.

### App bundle not found
Ensure `--app_directory=app` is passed to the BuildBuddy server in `start_buildbuddy.js`.

### Tests can't find app.js
Make sure the `app_bundle_js` genrule uses:
```bazel
cmd = "cp $(location //app:app_bundle)/app.js $@",
```
Not just `cmd = "cp $< $@"`, since `app_bundle` is a directory.

## Best Practices

1. **Keep tests focused**: Each test file should test a specific feature or user flow
2. **Use unique ports**: If creating multiple test suites, use different ports to avoid conflicts
3. **Clean state**: Each test should work with a fresh database (the server restart handles this)
4. **Meaningful assertions**: Test user-facing behavior, not implementation details
5. **Reuse configuration**: Put common config in `test_environment.js` to share across test files

## Example: Testing with Data

If you need to pre-populate data, modify `start_buildbuddy.js` to seed the database or create files before starting the server. Alternatively, use API calls in your test's `beforeEach` hook.
