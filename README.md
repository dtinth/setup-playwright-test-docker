# setup-playwright-test-docker

Running `npx playwright install --with-deps` in GitHub Actions normally takes about a minute — it downloads and installs browsers and their system dependencies from scratch on every run.

This action is a drop-in replacement that is 2–3x faster. It starts a [Playwright](https://playwright.dev/) server using a [pre-built Docker image](https://playwright.dev/docs/docker) and [configures the `PW_TEST_CONNECT_*` environment variables](https://playwright.dev/docs/docker#remote-connection) so your tests connect to it automatically — no changes to your test code required.

## Usage

```yaml
steps:
  - uses: dtinth/setup-playwright-test-docker@main

  - run: npx playwright test
```

No other changes needed. The action sets `PW_TEST_CONNECT_WS_ENDPOINT` and `PW_TEST_CONNECT_EXPOSE_NETWORK` in the environment, which Playwright picks up automatically to run tests against the remote server.

## Inputs

| Input | Description | Default |
|---|---|---|
| `version` | Playwright version | auto-detected from `@playwright/test` |
| `port` | Port to expose the Playwright server on | `43424` |
| `container-name` | Docker container name | `playwright` |

The `version` is auto-detected from the `@playwright/test` package installed in your project. You can override it explicitly if needed.

## How it works

1. Pulls `mcr.microsoft.com/playwright:v<version>-noble` and starts a container running `playwright run-server`.
2. Sets [`PW_TEST_CONNECT_WS_ENDPOINT`](https://github.com/microsoft/playwright/blob/837d8eb434baef2df54e234faa837e1c846ab50c/packages/playwright/src/index.ts#L551)`=ws://127.0.0.1:<port>/` so Playwright connects to the remote server instead of launching a local browser.
3. Sets [`PW_TEST_CONNECT_EXPOSE_NETWORK=*`](https://github.com/microsoft/playwright/blob/837d8eb434baef2df54e234faa837e1c846ab50c/packages/playwright/src/index.ts#L558) (undocumented) to expose the runner's network to the browser inside Docker. Playwright routes matching browser socket connections through a [server-side SOCKS bridge](https://github.com/microsoft/playwright/blob/837d8eb434baef2df54e234faa837e1c846ab50c/packages/playwright-core/src/server/utils/socksProxy.ts) and tunnels them [over the existing Playwright connection](https://github.com/microsoft/playwright/blob/837d8eb434baef2df54e234faa837e1c846ab50c/packages/playwright-core/src/server/socksInterceptor.ts) to sockets opened on the client — so the browser can reach services running on the runner (e.g. your dev server).
4. Waits for the server to be ready before continuing.

## Requirements

- Docker must be available on the runner (it is on `ubuntu-*` runners by default).
- `npx` must be available (for the `wait-on` readiness check).
