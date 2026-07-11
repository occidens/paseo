# Building the desktop app for Linux arm64

CI (`.github/workflows/desktop-release.yml`) only builds Linux desktop artifacts on
`ubuntu-22.04` with `--x64`. There is no arm64 Linux job, so `paseo.sh/download` and
the GitHub releases never ship a `linux-arm64` build. Nothing prevents building one
locally — `electron-builder.yml`'s `linux` target has no `arch` restriction, Electron
ships official `linux-arm64` prebuilts, and both native deps (`node-pty`,
`sherpa-onnx-node`) publish `linux-arm64` prebuilt binaries. Building natively on
arm64 hardware just works; nobody has wired up the CI job.

## Prerequisites

Install once per machine:

```bash
sudo dnf install -y libxcrypt-compat rpm-build
```

- **`libxcrypt-compat`** — electron-builder's bundled Ruby/`fpm` (used to build `.deb`
  and `.rpm`) links against `libcrypt.so.1`. Modern Fedora ships `libxcrypt` without
  the legacy compat symlink by default, so `fpm` fails with:
  `ruby: error while loading shared libraries: libcrypt.so.1: cannot open shared object file`
- **`rpm-build`** — provides `rpmbuild`, required for the `.rpm` target specifically.
  Not installed by default even on Fedora desktop.
- Node.js is managed via `mise`, which reads the pin from `.tool-versions` (currently
  `nodejs 22.20.0`) — run `mise install` in the repo root if you don't already have it.
  Don't add a `[tools]` block to `.mise.toml` to pin Node; it silently overrides
  `.tool-versions` and drifts from the version CI builds with.

## Build

```bash
npm ci
npm run build:relay
npm run build:desktop -- --linux --arm64
```

The `build:relay` step is not optional when the checkout has been updated since the
last build. `npm ci` wipes `node_modules` but never touches the per-package `dist/`
directories, and those are gitignored, so they survive any pull or rebase. Root
`build:desktop` starts with `build:app-deps:clean`, which rebuilds highlight,
protocol, and client — but not relay, whose declarations only get regenerated much
later inside the desktop package's own `build:server:clean`. The client therefore
typechecks against whatever stale `packages/relay/dist/*.d.ts` is lying around. When
relay's `Transport` interface changed between 0.1.105 and 0.3.0-beta.2, that surfaced
as a baffling error in a file neither release had touched:

```
src/daemon-client-relay-e2ee-transport.ts(116,32): error TS2345:
  Argument of type 'RelayTransportMessage' is not assignable to parameter of type 'string | ArrayBuffer'.
```

Relay has no workspace dependencies, so building it first is always safe. For a
guaranteed-clean build, `rm -rf packages/*/dist packages/*/*.tsbuildinfo` first —
then relay must be built before anything else, or the client build fails on a
missing module instead.

This runs the Expo web export, builds the server/CLI TypeScript workspaces, then
invokes electron-builder for all four Linux targets. Output lands in
`packages/desktop/release/`:

- `Paseo-arm64.AppImage` — portable, no install required, works on any distro
- `Paseo-<version>-arm64.deb`
- `Paseo-<version>-aarch64.rpm`
- `Paseo-<version>-arm64.tar.gz`

To re-run just the packaging step (e.g. after fixing one of the prerequisites above,
without repeating the Expo/TypeScript build):

```bash
cd packages/desktop
../../node_modules/.bin/electron-builder --config electron-builder.yml --linux deb rpm --arm64
```

## Cross-compiling from x86_64 instead

Not recommended unless arm64 hardware isn't available. Electron and the app's own
TS/JS code are architecture-independent to package, but `node-pty` and
`sherpa-onnx-node` resolve their prebuilt binary based on the _installing_ Node
process's actual `process.arch` — there's no npm flag that reliably overrides this
for both packages at once. The practical route is running the whole `npm ci` +
build inside an emulated arm64 container, e.g.:

```bash
docker run --rm --platform linux/arm64 -v $PWD:/work -w /work node:22 \
  bash -c "npm ci && npm run build:desktop -- --linux --arm64"
```

(requires `binfmt_misc`/`qemu-user-static`, e.g. via
`docker run --privileged --rm tonistiigi/binfmt --install arm64` on Linux hosts).
Expect several times slower than a native build due to QEMU emulation overhead.
