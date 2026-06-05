# hello-pear-bare

> Pear Hello World for Bare CLI with `pear-runtime`

End-to-end boilerplate for embedding [pear-runtime](https://github.com/holepunchto/pear-runtime) into a [Bare](https://github.com/holepunchto/bare) CLI with a terminal UI, peer-to-peer OTA update support and standalone builds.

- Terminal UI built with [`@holepunchto/bare-tui`](https://github.com/holepunchto/bare-tui) (a Bubble Tea-style framework for Bare)
- Peer-to-Peer Over-the-Air updates
- Bare worker process via `PearRuntime.run(...)`, with its IPC routed into the UI
- Cross-platform standalone distributables via [`bare-build`](https://github.com/holepunchto/bare-build)

## Table of Contents

- OS Support
- Requirements
- Development
  - Install
  - Create an upgrade link
  - Start
- Architecture
  - Terminal UI
  - Updates
  - Workers
- Peer-to-Peer Deployments
- Scripts
- Project Structure
- Troubleshooting

## OS Support

- **macOS** — arm64, x64
- **Linux** — arm64, x64
- **Windows** — arm64, x64

## Requirements

- `npm` via [Node.js](https://nodejs.org/)
- [pear](https://docs.pears.com/) - `npx pear`

## Development

### Install

```sh
npm install
```

### Create an upgrade link

This template expects `package.json` to contain a valid `pear://` link in the `upgrade` field. If it still contains the placeholder `pear://<YOUR_KEY_HERE>`, startup will fail with `INVALID_URL`.

Create a link with [`pear touch`](https://docs.pears.com/reference/cli.html#pear-touch-flags-channel):

```sh
pear touch
```

Copy the generated `pear://...` link into the `upgrade` field in `package.json`.

### Start

Start app in development mode:

```sh
npm start
```

This launches the terminal UI (a spinner header and a live feed of worker IPC).
Press <kbd>q</kbd> or <kbd>Ctrl+C</kbd> to quit; the terminal is restored on exit.

By default this repo starts with `--no-updates` in development to avoid local dev binaries being swapped while you iterate.

Enable updates for local flow testing:

```sh
npm start -- --updates
```

Send a custom worker message:

```sh
npm start -- --message "hello from dev"
```

## Architecture

`bin.js` is the happy path — your app, a terminal UI. `lib/pear-cli.js` holds the
boilerplate: storage resolution, pear-runtime construction, the OTA updater,
swarm replication, signal handling and teardown — and it drives your UI as a
[bare-tui](https://github.com/holepunchto/bare-tui) program. You rarely need to
touch it.

`createPearCli(pkg, opts)` returns a handle with `start(build)` and
`run(script, opts)`. `build({ run, send, flags, pear, swarm, store, appName, dir })`
returns your model — a bare-tui`{ init?, update, view }` object that the wrapper
runs as the UI.

```js
const createPearCli = require('./lib/pear-cli')
const { quit, key } = require('bare-tui')
const pkg = require('./package.json')

class App {
  constructor({ run, flags }) {
    this.worker = run('./workers/main.js') // worker output arrives as Msgs
    this.worker.write(Buffer.from(flags.message || 'hello'))
    this.last = ''
  }
  update(msg) {
    if (msg.type === 'worker') this.last = String(msg.data).trim()
    if (key.matches(msg, 'q', 'ctrl+c')) return [this, quit]
    return [this, null]
  }
  view() {
    return `worker says: ${this.last}\n\npress q to quit`
  }
}

const cli = createPearCli(pkg, {
  flags: [['--message <text>', 'message sent to worker IPC stream']],
  handlers: {
    onUpdate: async ({ updater }) => {
      await updater.applyUpdate() // required — decide when to apply & restart
    }
  }
})

cli.start(({ run, flags }) => new App({ run, flags }))
```

See [`bin.js`](bin.js) for the full demo model.

### Terminal UI

The UI is a [bare-tui](https://github.com/holepunchto/bare-tui) program — The Elm
Architecture (`init` / `update` / `view`) with components, styling and layout.
`cli.start(build)` builds your model and runs it; the `Program` owns the screen
(raw mode, alternate screen, input decoding) and always restores the terminal on
exit, even on error.

Because the UI owns the screen, the wrapper never prints to stdout directly.
Instead it routes everything into your `update` as messages:

| Message                                                                | Source                                                          |
| ---------------------------------------------------------------------- | --------------------------------------------------------------- |
| `{ type: 'log', level, line }`                                         | the wrapper / updater (version banner, update progress, errors) |
| `{ type: 'worker', stream, script, data \| code }`                     | a spawned worker's `stdout` / `stderr` / `ipc` / `exit`         |
| `{ type: 'key' }`, `{ type: 'resize', ... }`, `{ type: 'mouse', ... }` | terminal input (from bare-tui)                                  |

`send(msg)` (also passed to `build`) lets you inject your own messages from
anywhere. To learn the framework — components (`list`, `table`, `textinput`, …),
`style`, testing — see the [bare-tui docs](https://github.com/holepunchto/bare-tui).

### Event handlers

Handlers are passed to the constructor via `handlers`. One is required, the rest
are optional and fall back to logging:

| Handler                            | Required | Fires on                                                                        |
| ---------------------------------- | -------- | ------------------------------------------------------------------------------- |
| `onUpdate({ updater, pear, ... })` | **yes**  | a new version is ready — you decide whether/when to `applyUpdate()` and restart |
| `onUpdating(ctx)`                  | no       | an update download starts                                                       |
| `onUpdatingDelta(delta, ctx)`      | no       | update download progress                                                        |
| `onConnection(connection, ctx)`    | no       | a swarm peer connects (replication already runs; this augments it)              |
| `onError(err, ctx)`                | no       | a `pear-runtime` error                                                          |

`onUpdate` is mandatory because applying an update and restarting is a product
decision every CLI must make — the framework will not guess it for you. Leaving
it out throws at startup.

### Updates

Updates are handled through `pear-runtime` and the configured `upgrade` link in `package.json`.

Per-run disable updates:

```sh
npm start -- --no-updates
```

### Workers

`run('./workers/main.js')` spawns a Bare worker over `pear-runtime`. By default
its `stdout` / `stderr` / IPC / `exit` are routed into your model as
`{ type: 'worker', stream, script, data | code }` messages, so the UI reacts to
them without corrupting the screen — the `ipc` stream is the worker's channel and
is usually where your app logic lives. Pass `{ onData, onStdout, onStderr, onExit }`
to intercept a stream directly instead.

## Peer-to-Peer Deployments

Set the `upgrade` field in `package.json` to your distribution drive link, then follow the default flow from section 4 onward:

[hello-pear-electron: 4. Build Deployment Directory and onward](https://github.com/holepunchto/hello-pear-electron#4-build-deployment-directory-)

## Scripts

- `npm start` - run the Bare CLI in dev mode (`bare bin.js --no-updates`)
- `npm test` - run `brittle-bare` tests
- `npm run lint` - run prettier check and lunte
- `npm run format` - format repository with prettier
- `npm run make` - auto-detect host OS/arch and run matching build target
- `npm run make:darwin-arm64` - build standalone to `out/darwin-arm64`
- `npm run make:darwin-x64` - build standalone to `out/darwin-x64`
- `npm run make:linux-arm64` - build standalone to `out/linux-arm64`
- `npm run make:linux-x64` - build standalone to `out/linux-x64`
- `npm run make:win32-arm64` - build standalone to `out/win32-arm64`
- `npm run make:win32-x64` - build standalone to `out/win32-x64`

## Project Structure

- `bin.js` - CLI entrypoint — your terminal UI (a bare-tui model) and event handlers
- `lib/pear-cli.js` - runtime/updater/swarm/teardown boilerplate that runs your UI (rarely edited)
- `workers/main.js` - Bare worker example
- `scripts/make.js` - platform/arch build target selector
- `test/index.js` - brittle-bare tests

The terminal UI framework lives in its own package,
[`@holepunchto/bare-tui`](https://github.com/holepunchto/bare-tui) (a dependency).

## Troubleshooting

- `INVALID_URL: Invalid URL 'pear://<YOUR_KEY_HERE>'` means the placeholder `upgrade` link in `package.json` has not been replaced. Run `pear touch`, then put the generated `pear://...` link in `package.json`.
- If updates do not trigger, verify `package.json` contains a valid `upgrade` Pear link and that peers are seeding the target drive.
- If `npm run make` fails on unsupported hosts, run a specific `make:<platform>-<arch>` script or build on a supported host.
- This template does not implement app-level data persistence; it is a minimal CLI + updater example.
