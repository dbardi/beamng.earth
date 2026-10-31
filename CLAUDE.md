# CLAUDE.md

BeamNG.Earth streams the real Earth into BeamNG.drive. It has three parts that must agree with each other:
- a Python world server, `server/earthd.py` ("earthd")
- a Lua game-side mod, `mod/beamng_earth/`
- a required Rust extension module, `native/earthfast`

`launcher.py` ties them together for players. The main docs are `README.md`, `server/README.md` (flags, protocol
tables, subsystems) and `docs/ARCHITECTURE.md` (design and in-game results).

## Commands

```sh
python3 install.py                       # copy the mod into the BeamNG user folder, make .venv, install server/requirements.txt
python3 start.py                         # run the server with the recommended feature flags (uses .venv)
python3 server/earthd.py [flags]         # run the server by hand (listens on 127.0.0.1:47613; flags in server/README.md)
python3 launcher.py [--no-game]          # what the packaged app runs: syncs the mod, starts the server and the game

# Python tests (unittest, no pytest config). BNGE_SKIP_DRIVE=1 skips the long drive test.
python3 -m unittest discover -s server/tests -v
python3 tools/run_tests.py [test_roads test_dem ...]   # one process per module, parallel, per-module timeout
python3 -m unittest server/tests/test_roads.py         # one module, from the repo root
cd server/tests && python3 -m unittest test_geodesy.GeodesyTest.test_round_trip   # one test

# Lua tests (LuaJIT, offline, engine stubbed by mod/tests/stubs.lua and testlib.lua)
luajit mod/tests/test_governor.lua
cd mod/tests && for f in test_*.lua; do luajit $f; done

# Rust kernels
python3 tools/build_native.py            # builds native/earthfast and writes server/earthfast.abi3.so (--windows: the .pyd via cargo-zigbuild)
cd native/earthfast && cargo test --release

# Packaging
python3 tools/package.py [--beammp]      # source zip (and the BeamMP server zip) into dist/
python3 tools/build_app.py               # this system's AppImage / .exe (PyInstaller; run tools/build_bootloader.py first for the .exe)
```

There is no linter or formatter config. CI (`.github/workflows/release.yml`) only builds the app and runs
`--help`; it runs none of the tests, so run the relevant suites locally. Some tests need the network (for example
`test_e2e.py` downloads real elevation tiles), and some skip themselves without LuaJIT or game files
(`test_modsim.py`, `test_mesh.py`). `test_advance_sign_layout.py` also needs `fontTools`, which is not in
`server/requirements.txt`.

## Architecture

- **Floating origin.** The server holds the real WGS84 Earth in float64. The game only sees a local East-North-Up
  frame of a few km around an *anchor*. Every tile is sent already transformed into the current anchor's frame.
  Near the anchor's edge the mod *rebases*: it picks a new anchor, applies one rigid transform to everything, and
  rebuilds collision once. Many messages carry `anchorId`, and "anchor", "rebase", "commit" and "held" are core
  concepts on both sides.
- **Transport.** The mod (`net.lua`) talks to earthd over a localhost TCP socket with JSON messages
  (`PROTOCOL = 4` in both `server/earthd.py` and `core.lua`). Bulk data is not sent over the socket. The server
  writes files (heightmaps, `.cdae` shapes, DDS textures, JSON tiles) under
  `<BeamNG user dir>/temp/beamngEarth/<sessionId>/` with versioned names, because the engine caches by path. The
  messages name those files.
- **Server side.**
  - `earthd.py`: the asyncio server, session state and the message `dispatch`.
  - Subsystem modules: `roads.py`, `buildings.py`, `grading.py`, `farfield.py`, `imagery.py`, `vegetation.py` and others.
  - `blockworker.py`: process pools for the physics blocks, road tiles and far tiles, so building work doesn't
    hold the main interpreter.
  - `netfetch.py`: every download goes through it (rate limits, failover, offline mode).
  - `mapdata.py`: Overture map data through DuckDB.
  - `cache.py`: the download cache.
- **Geotiles.** Much content (vegetation, road tiles, far buildings) is built in a global zoom-17 Web-Mercator
  grid in its own frame (`geotiles.py`, `geotiles.rs`, mod `geotile.lua`). It is placed into the current anchor and
  kept across rebases. Only the per-anchor "skins" (draped surfaces) are re-sent. `server/README.md` (`--geo-tiles`)
  explains it.
- **Mod side.** `core.lua` coordinates: its `onMessage` routes server messages and `onUpdate` drives the frame.
  Work is spread over feature modules (`roads.lua`, `farfield.lua`, `trees.lua`, `sea.lua`, ...). Pure-logic
  modules (`governor`, `follow`, `rebase`, `lattice`, `geodesy`, `edgeguard`) avoid engine calls, so they can be
  tested under LuaJIT.
- **Rust (`native/earthfast`, pyo3).** It holds the hot geometry and texture kernels, and `server/fast.py` wraps it.
  It is required: the Python fallbacks were removed, whatever the comments at the top of `lib.rs` and `Cargo.toml`
  still say. `native/serverlist` is a separate small service that lists BeamNG.Earth BeamMP servers. `beammp/Server`
  is the BeamMP server plugin.
- **Feature flags.** New features land behind earthd flags, defaulting to off, until an in-game check passes.
  `RECOMMENDED` in `start.py` (source runs) and in `launcher.py` (the packaged app) turns the shipped set on. Keep
  the two lists equal; `test_gasstations.py` checks this.

## Cross-language invariants (the tests enforce these)

- **Protocol.** `server/tests/test_protocol_agreement.py` keeps one table of every message and its fields. It
  checks this against the server's real session output and against the Lua sources it parses (`net.send` builders,
  `msg.X` reads), along with constants both sides share. When you add or change a message field, update that table
  and the protocol tables in `server/README.md`.
- **Lua and its Python model.** `server/tests/modsim.py` is a Python model of the mod's governor, follow, rebase,
  farfield and edgeguard logic. `test_modsim.py` runs the real Lua under LuaJIT on the same schedules
  (`server/tests/lua/*.lua`) and requires identical traces frame by frame. Change both together.
- **Defaults.** `test_v131_default.py` compares a default `earthd.Server` against `fixtures/v131_golden.json`. A
  change to default behavior fails it; put new behavior behind a flag. Regenerate the fixture only as its docstring
  describes.
- **Native module version.** `native/earthfast/src/lib.rs` exports `VERSION` (`m.add("VERSION", N)`), and
  `server/fast.py` has `MIN_VERSION`. Bump both when you add or change a kernel signature. The built binaries are
  committed (`server/earthfast.abi3.so` for Linux x86-64, `server/earthfast.pyd` for Windows) and source installs
  load them, so rebuild and commit them with the Rust change.
- **Generated mod assets.** The far-tile material file (`main.materials.json`) and `far_palette.png` in the mod
  are generated by `python3 server/farfield.py`. `test_farfield.py` checks that the shipped copies match the code.

## Releases and updates

- `VERSION` is the release version. A push to `main` whose `VERSION` has no GitHub release yet builds and publishes
  `v<VERSION>` publicly. Every push to `dev` publishes a development pre-release. Don't change `VERSION` or push
  to `main` / `dev` unless asked.
- The in-game updater (`server/selfupdate.py`, `server/updates.py`) checks the repository named in
  `server/updates.py` `REPO`, which is the upstream `TecnaGamer/BeamNG.Earth`, not this fork.

## In-game verification

BeamNG.drive has a built-in MCP server (Options > Advanced > MCP, `http://127.0.0.1:29292/mcp`).
`tools/mcp.py lua '<code>'` runs game Lua, and `tools/mcp.py call screenshot` takes screenshots. `tools/drive.py`,
`tools/benchmark.py` and `probes/` build on it. Offline tests stub the engine, and every engine behavior they assume
is marked for in-game checking (`mod/tests/stubs.lua`).

## Conventions

- Existing code, comments and identifiers use British spelling (`centre`, `metres`, `colour`). Some of these are
  message fields or file names shared across Python, Lua and tests (for example `centres`). Don't rename existing
  identifiers. New prose follows the user's preferences.
- Comments record *why*, often with the date and the report or measurement behind a choice. Keep that context
  when editing nearby code.

## Agent skills

### Issue tracker

GitHub Issues on this fork, `dbardi/beamng.earth`. See `docs/agents/issue-tracker.md`.

### Triage labels

The five default labels: `needs-triage`, `needs-info`, `ready-for-agent`, `ready-for-human`, `wontfix`. See `docs/agents/triage-labels.md`.

### Domain docs

Single-context: `GLOSSARY.md` and `docs/adr/` at the repo root. See `docs/agents/domain.md`.
