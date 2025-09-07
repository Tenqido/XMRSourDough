# XMR Sourdough: MIT-licensed RandomX miner and minimal Stratum client

XMR Sourdough is an MIT-licensed Monero miner concept and codebase-in-progress. The aim is simple and practical: provide a clean, permissively licensed RandomX CPU miner plus a minimal Stratum client that applications can embed or ship alongside without inheriting GPL obligations. The project takes a conservative, engineering-first approach focused on clarity, consent, and maintainability.

XMR Sourdough is designed to be small, auditable, and friendly to upstream integrations such as Godot 4.4.1 front ends, desktop apps, and hardware appliances. The name reflects the intent to be a starter that helps new projects rise.

## Why this exists

Monero’s network uses RandomX, which is intentionally CPU-friendly. The most mature miners are GPL-licensed. That is good for open source, but it can be difficult if you need to ship a closed-source game or appliance and want to include a miner component with clear consent and transparent controls. A permissive MIT codebase lowers friction for experimentation and adoption while remaining ethical and user-centered.

The goal is not to outcompete XMRig day one. The goal is to establish a clean, modern baseline you can build on: a standards-compliant Stratum client, a correct RandomX pipeline, reasonable defaults, and a well-documented surface for GUI tooling.

## Scope and non-goals

1. Scope: CPU-only RandomX mining with a minimal Stratum client, basic configuration, and optional read-only HTTP status for UIs.
2. Scope: permissive licensing end to end. No GPL libraries. No ambiguous headers. RandomX used via its BSD 3-Clause implementation.
3. Scope: production discipline. Clear config, logs, and safe defaults. No stealth features. No persistence tricks.
4. Non-goal: GPU mining. RandomX is CPU-optimized and GPU support is out of scope.
5. Non-goal: feature sprawl. Target a tight core that other tools can extend.

## Design principles

1. Correctness before speed. Submit valid shares reliably under varying pool targets and reconnect conditions.
2. Transparency by default. Every action is visible in logs and status. No hidden behaviors.
3. Respect for the user. Mining is opt-in and explicit. Resource usage is controllable.
4. Portability. Build cleanly on Linux, Windows, and macOS using CMake and modern C++.
5. Integratability. Provide a simple HTTP summary to make GUI integration trivial.

## Architecture overview

At a high level the miner is a set of small modules.

1. Stratum client: TCP or TLS socket, subscribe and authorize, consume set\_job and set\_target, submit shares, handle backoff and reconnect.
2. Job manager: holds the current job, target, seed hash, and a monotonically increasing job version for worker synchronization.
3. RandomX engine: thin wrapper around the BSD RandomX library that manages cache, dataset, and one VM per worker thread.
4. Worker pool: a configurable number of CPU workers that iterate nonce ranges, compute RandomX hashes, and enqueue valid shares.
5. Submit queue: non-blocking handoff for shares from workers to the Stratum client.
6. Stats and telemetry: moving-average hashrates and counters for accepted, rejected, stale. Optional HTTP endpoint for GUIs.
7. Config and logging: TOML or JSON configuration, console logs, and a rotating file log.

## What exists to build upon

1. RandomX library: BSD-licensed reference implementation that can be statically linked. This is the heart of a permissively licensed miner.
2. Prior art on Stratum: old MIT-licensed CryptoNight miners and open references illustrate the message flow and message formats. While their hashing cores are obsolete, the Stratum scaffolding patterns remain useful.
3. Lightweight MIT utilities: header-only JSON parsers and tiny HTTP servers make it easy to stay permissive without heavy dependencies.

## Language and toolchain

1. C++20 with CMake for reproducible cross-platform builds.
2. Linux: gcc or clang. Windows: MSVC. macOS: AppleClang.
3. Release builds use optimization and link-time optimization. Optional PGO later.
4. No GPL dependencies. RandomX BSD only. JSON and HTTP libraries under MIT or similar.

## Folder layout

```
miner/                          # repo root
├─ CMakeLists.txt               # build
├─ LICENSE                      # MIT
├─ README.md                    # this file
├─ third_party/
│   └─ randomx/                 # BSD 3-Clause RandomX (submodule or vendor)
├─ include/                     # public headers (kept small)
├─ src/
│   ├─ main.cpp                 # entrypoint and wiring
│   ├─ config/Config.*          # load config, defaults, validation
│   ├─ logging/Log.*            # console and file logging
│   ├─ stratum/Client.*         # TCP/TLS, JSON-RPC messaging, retries
│   ├─ stratum/Message.*        # parse and build Stratum JSON
│   ├─ core/Job.*               # job struct and versioning
│   ├─ core/JobManager.*        # thread-safe current job
│   ├─ core/Worker.*            # mining threads
│   ├─ core/SubmitQueue.*       # share handoff to network thread
│   ├─ core/Stats.*             # counters and hashrate windows
│   ├─ rx/Engine.*              # RandomX wrapper
│   ├─ api/Http.*               # optional read-only HTTP summary
│   └─ sys/{HugePages,NUMA,MSR}.* # performance helpers behind flags
```

## MVP feature checklist

1. Single pool connection over TCP. Subscribe, authorize, receive jobs, submit shares.
2. CPU RandomX in full dataset mode with JIT. Cache and dataset lifecycle handled correctly.
3. N worker threads with auto detection or explicit count.
4. Basic CLI flags and a simple config file for pool, wallet, worker name, and thread count.
5. Console stats showing 1 s, 30 s, and 1 m hashrates plus accepted and rejected shares.
6. Clean shutdown and reconnect handling.

## Roadmap

1. Phase 1: TLS support, reconnect and backoff strategies, and failover pool list.
2. Phase 2: read-only HTTP status at /summary for GUIs. Optional plain text status for scripts.
3. Phase 3: performance features such as huge pages, NUMA pinning, adaptive nonce batching, and optional MSR tweaks behind explicit flags.
4. Phase 4: cross-platform release builds and CI, signing on Windows, and reproducible build instructions.
5. Phase 5: stagenet mode for safe testing, improved diagnostics, and a small pool compatibility matrix.

## Configuration and flags

Configuration will be minimal and explicit. A single JSON or TOML file plus a few CLI overrides is sufficient.

1. Pool settings: url, tls, wallet, worker name, password hint if required by the pool.
2. Miner settings: threads, dataset mode full or light, huge pages on or off, JIT enabled by default.
3. Logging: log level and file rotation.
4. API: optional HTTP host and port for read-only status.

A small example will be added once code lands. The goal is to keep fields obvious and documented inline.

## Performance and correctness plan

1. Correctness: focus on submitting valid shares across common pools, handling difficulty changes, and avoiding duplicate submissions on job switches.
2. Hashrate baselines: compare to XMRig on the same host to ensure the RandomX pipeline is configured properly. Expect to be within a reasonable margin early on. Close gaps incrementally.
3. Memory: allocate dataset once when the seed hash changes. Avoid per-hash allocations. Reuse per-thread buffers.
4. CPU: default to one VM per worker thread. Allow manual thread count and affinity for advanced users.
5. System features: huge pages can materially improve RandomX performance. Provide guarded helpers to request them. Document permissions needed on each OS. Make all such features opt-in.

## Security, ethics, and user consent

1. Opt-in mining: the miner will never start mining without explicit user action. No background mining. No hidden startup entries.
2. Clear user controls: start, stop, and exit are obvious. CPU limits and thread controls are respected.
3. No stealth: no process injection, no watchdogs that respawn without consent, no process killers, and no anti-AV behaviors.
4. Privacy: no telemetry by default. If optional metrics are ever proposed they will be documented and off by default.
5. Documentation: the README will include plain-language explanations of resource use, heat, and power considerations. Users should understand what mining means.

## Godot integration guide

The intended integration pattern is clean and simple so game developers can offer mining features only when players explicitly opt in.

1. One process model: ship XMR Sourdough as a separate binary under MIT. Your Godot 4.4.1 game starts and stops it using `OS.execute` or `OS.execute_with_pipe`. No GPL code is included or redistributed.
2. Status UI: enable the miner’s HTTP summary endpoint. Your Godot UI polls `/summary` with `HTTPRequest` and renders hashrate, shares, and uptime. This keeps the GUI responsive and the miner independent.
3. Configuration: store the miner config under your Godot `user://` directory. Provide a settings screen that edits pool, wallet, and resource limits. Write out the config and restart the miner when needed.
4. UX patterns: show a consent dialog that explains power and heat effects. Provide a Start Mining button, CPU thread slider, and a visible Stop button. Persist the user’s choice. No auto start.
5. Distribution: keep your Godot project under any license you prefer. XMR Sourdough remains MIT. The two are loosely coupled over a local API or process pipes.

## Appliance and hardware notes

The permissive license is also useful for small devices and appliance builders who want to integrate CPU mining with clear consent.

1. House heaters and combined workloads: RandomX favors CPUs and large memory. A permissive codebase lets you embed a miner in firmware or an appliance application without GPL constraints, provided you satisfy the MIT terms.
2. Controls and safety: such deployments need explicit thermal management, rate limiting, and watchdogs that are honest and user-visible. That is outside the core scope here but the codebase will remain modular so integrators can add the controls they need.
3. Updates: appliance builders should track RandomX releases and rebuild. The project will aim to make upgrades obvious and low risk.

## Compatibility and pools

The Stratum client targets common Monero pool implementations. The MVP will ship with a short list of tested pools and the exact Stratum options used. Over time a compatibility matrix will document pool quirks, TLS defaults, and recommended timeouts. Stagenet endpoints will be listed for safe functional tests that do not affect mainnet payouts.

## Building and contributing

Builds use CMake and a minimal dependency set.

1. Prerequisites: a modern C++ toolchain, CMake, and system packages required by RandomX.
2. Clone with submodules: the RandomX dependency lives under third\_party.
3. Standard steps: create a build directory, configure with CMake, build in Release mode.
4. First run: create a sample config, connect to a small pool or stagenet, and observe the first accepted share.
5. Issues and PRs: contributions are welcome that preserve the project goals. Keep changes small and well documented.

Please follow a few guardrails for contributions.

1. Keep the code permissive only. Do not add GPL code or headers.
2. Respect the security and ethics policy. No stealth, no persistence tricks.
3. Discuss large features in an issue before opening a PR.
4. Prefer clear, well-commented code over cleverness.

## Road to 1.0

A simple path to a stable release is planned.

1. 0.1: MVP with TCP, CPU RandomX full dataset, multi-thread workers, console stats, and clean shutdown.
2. 0.2: TLS and basic reconnects with exponential backoff. Failover pool list.
3. 0.3: read-only HTTP summary endpoint for GUIs. Hashrate windows and counters.
4. 0.4: performance features such as huge pages and optional NUMA pinning. Documented setup steps per OS.
5. 0.5: cross-platform CI, signed Windows binaries, reproducible build notes.
6. 1.0: documented pool compatibility matrix, stagenet quickstart, and stability under long runs.

## Testing strategy

Functional and stability testing gets priority over micro-optimizations during the early stages.

1. Functional tests: connect to stagenet and verify job handling, share submission, and difficulty changes. Ensure correct nonce handling on job switches.
2. Stability tests: long-running sessions with forced network interruptions, TLS on and off, and simulated pool restarts.
3. Performance baselines: compare to XMRig on the same host to check for configuration errors. Use the comparison as a sanity check, not a target for cloning.
4. Integration tests: exercise the HTTP summary endpoint, then validate against a Godot demo UI that starts and stops the miner and displays status.

## License

This project is licensed under MIT. The RandomX library included as a submodule or vendored under third\_party is BSD 3-Clause. No GPL code will be added to the repository. Check each third-party component for its own license file.

## Security and disclosure

If you discover a security issue, please open a private contact channel described in SECURITY.md once it exists. Do not post exploit details in public issues. The project prefers responsible disclosure and rapid fixes.

## What this project is not

1. It is not a stealth miner.
2. It is not a drop-in replacement for XMRig on day one.
3. It is not a GPU miner.
4. It is not a place for license bait. The goal is clarity and permission for builders.

## How to use this in a Godot 4.4.1 project

A typical pattern looks like this.

1. Place the miner binary next to your game executable in a subfolder. Keep configs under `user://`.
2. On Start Mining, write the config file and call `OS.execute` to launch the miner. Hide the console window on Windows.
3. Poll the miner’s `/summary` every few seconds with `HTTPRequest`. Render hashrate, shares, and runtime in your UI.
4. Provide Stop Mining and Exit buttons. On stop, send a polite termination signal and wait for the process to shut down.
5. Document in-game what mining does. Require explicit consent each time the user toggles mining on a new session or after major updates.

This pattern keeps licenses clean and creates a pleasant user experience. The miner remains a focused binary. Your game remains your game.

## Call for collaborators

If you want an MIT-licensed RandomX miner to exist, this is a good moment to help shape it. Useful first contributions include a minimal Stratum client, a thin RandomX wrapper, and a small HTTP summary server. Careful logging and a simple config reader go a long way. If you are building a Godot front end, a small status panel and start or stop control makes a great companion demo.

Thank you for reading. If XMR Sourdough helps you bake mining into respectful, user-consented applications, then it is doing its job.
