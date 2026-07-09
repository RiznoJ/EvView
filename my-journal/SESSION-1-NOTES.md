# Session 1 — 2026-07-08

## Goal for today
Get the RuView (my fork: EvView) WiFi sensing dashboard running on my Mac (M4 Pro, macOS Sequoia) using **fake/simulated data** — no ESP32 hardware yet, no Docker installed. Just wanted to see the live dashboard in a browser.

## What we did, step by step

1. **Installed Rust.**
   The dashboard's backend server is written in Rust, a programming language, and my Mac didn't have the Rust build tools installed. We installed them using `rustup`, the official Rust installer, by running a script from `https://sh.rustup.rs`. This only installed files into my own user folder (`~/.cargo` and `~/.rustup`) — no admin password, no system-wide changes.
   - **Tools installed:** `rustc` (the Rust compiler — turns Rust source code into a runnable program), `cargo` (Rust's build tool/package manager — like `npm` for Node.js, downloads dependencies and builds the project), `rustup` (manages which Rust version is active).

2. **Downloaded the project's "submodules."**
   This repo is built out of several smaller repositories bundled inside it (things like `vendor/rufield`, `vendor/rvcsi`). These are called git submodules, and they don't get downloaded automatically when you clone/fork a repo — they start as empty placeholder folders. We ran `git submodule update --init --recursive` to fetch all 8 of them from GitHub. This didn't change any of my own files, it just filled in folders that were already there but empty.

3. **Built the sensing server.**
   Ran `cargo build -p wifi-densepose-sensing-server --no-default-features` from the `v2/` folder. This compiled the Rust code (and all its dependencies) into an actual runnable program. Took about 37 seconds. A few harmless warnings showed up (unused code, nothing broken).

4. **Started the server in "simulate" mode.**
   Ran the built program with a flag telling it to skip looking for real ESP32 hardware and just generate pretend sensing data instead:
   ```
   cargo run -p wifi-densepose-sensing-server --no-default-features -- --source simulate
   ```
   - First attempt used a flag (`--static-dir`) that doesn't actually exist in this build — got a clean error message, no harm done. Checked the program's `--help` output and found the real flag is `--ui-path`, which already defaults to the right folder (`../ui`) anyway, so we just left it out entirely on the next try.
   - Second attempt worked. The log confirmed: `Data source: simulated` and gave a URL to open.

## The one hiccup: a port conflict

When we tried to open the dashboard, the connection came back empty/broken. Turned out **an unrelated Python process already running on my Mac** was also listening on port 8080, but over a different network protocol (IPv6) than our new server (which only listens on IPv4). Browsers (and `curl`) sometimes try `localhost` over IPv6 first — which hit that unrelated Python process and got nothing back, instead of reaching our dashboard.

**Fix:** no code changes needed — just used the explicit address `http://127.0.0.1:8080/ui/index.html` instead of `http://localhost:8080/...`. That forces the connection over IPv4, straight to our server, sidestepping the other process entirely. We left that other Python process alone since it's unrelated to this project.

## Result

✅ Dashboard loaded successfully at `http://127.0.0.1:8080/ui/index.html`, showing live simulated sensing data updating in the browser.

## Notes for next time
- No files in the actual project code were changed today — only the Rust toolchain was installed on the Mac, and the git submodules were downloaded (both one-time setup steps, not project edits).
- One incidental file, `v2/Cargo.lock`, got modified locally as a side effect of building (it records exact dependency versions). Left uncommitted for now since it wasn't asked for.
- To restart the dashboard later: from the `v2/` folder, run
  `cargo run -p wifi-densepose-sensing-server --no-default-features -- --source simulate`
  then open `http://127.0.0.1:8080/ui/index.html`.
- When the ESP32 hardware arrives, we'll swap `--source simulate` for real hardware input.
