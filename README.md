# cheeky-loop-runner

A tiny Bash utility that runs a shell script inside a [tmux](https://github.com/tmux/tmux) session on a timed **and event-driven** loop: it starts your script, lets it run until either a fixed period elapses (30 minutes by default) **or your script touches a trigger file**, kills it by sending four `Ctrl-C`s to the session, then restarts it — forever, until you stop it.

It's handy for jobs that you want to run in fixed-length bursts and restart cleanly: long-running scrapers, training or fuzzing runs, watchers, anything that should be cycled rather than left running indefinitely. With the trigger file, the job itself can also say "recycle me now" without waiting out the timer.

## Features

- **Run anything** — point it at any shell script; no execute bit or shebang required.
- **Event-driven restarts** — your script touches a fixed trigger file and the runner recycles it immediately, no waiting for the timer.
- **"I'm done" signalling** — the body can touch a `process_ended` file to request a cleanup window followed by stopping the loop.
- **Linux-native watching** — uses `inotifywait` when available (no polling), and falls back to a dependency-free mtime poll otherwise.
- **Sandbox-friendly** — the script runs in the directory you invoked `cheeky-loop-runner` from, so relative paths inside your script resolve where you expect.
- **Clean restarts** — each cycle sends four `Ctrl-C`s, then restarts from a fresh prompt.
- **Two ways to stop** — `Ctrl-C` the runner, or touch the `process_ended` file (no need to find a PID).
- **Predictable sessions** — the tmux session is named after your script, so you can attach and watch it live.
- **Configurable period & wait** — default 30-minute cycles (or `0` for pure event-driven), plus a configurable grace/debounce wait before each restart.

## Requirements

- Bash 4+
- `tmux`
- `inotify-tools` *(optional, Linux)* — for instant, no-polling trigger detection. Without it the runner falls back to mtime polling.

On Debian/Ubuntu: `sudo apt install tmux inotify-tools`. On macOS: `brew install tmux` (no `inotifywait`; the poll fallback is used).

## Installation

Drop the `cheeky-loop-runner` script somewhere on your `PATH` and make it executable:

```bash
mkdir -p ~/.local/bin
cp cheeky-loop-runner ~/.local/bin/
chmod +x ~/.local/bin/cheeky-loop-runner
```

Make sure `~/.local/bin` is on your `PATH` (add `export PATH="$HOME/.local/bin:$PATH"` to your shell rc file if needed).

## Usage

```
cheeky-loop-runner <script.sh> [run_minutes]
```

From your sandbox directory:

```bash
cd ~/sandboxes/my-job
cheeky-loop-runner do_work.sh
```

Run with 45-minute cycles instead of the 30-minute default:

```bash
cheeky-loop-runner do_work.sh 45
```

Run in **pure event-driven mode** (no time ceiling — only the trigger file or process_ended file recycle/stop it):

```bash
cheeky-loop-runner do_work.sh 0
```

On startup it prints the resolved script path, the working directory, the tmux session name, the cycle length, the trigger and process_ended paths, and the watch backend in use.

### Triggering a restart from your script (event-driven)

The runner watches a fixed **trigger file** — by default `<script-name>.trigger` in the sandbox (e.g. `do_work.sh.trigger`). Whenever that file is created, touched, or written, the runner recycles the loop body right away rather than waiting out the timer.

So from inside your loop body (or any other process), just touch it when you want a restart — for example after finishing a unit of work, noticing new input, or detecting a config change:

```bash
touch do_work.sh.trigger      # or: : >> do_work.sh.trigger
```

The runner waits a configurable grace period (`CLR_WAIT_SECONDS`, default 240s / 4 min) so a burst of touches coalesces into a single restart and any final writes settle, then sends the `Ctrl-C`s and restarts. Only changes that happen *after* a run begins count, so a stale trigger file left over from a previous run won't cause a spurious restart.

Under the hood this uses `inotifywait` when it's installed (instant, no polling) and otherwise falls back to checking the trigger file's modification time every `CLR_POLL_SECONDS`. Force one or the other with `CLR_WATCH_BACKEND=inotify|poll`.

### Signalling that the process is done (event-driven)

The trigger file means "kill me and start over after the normal debounce." When instead your loop body has **finished its work and needs time to clean up first**, it can say so by touching the **process_ended file** — by default `<script-name>.process_ended` in the sandbox (e.g. `do_work.sh.process_ended`):

```bash
touch do_work.sh.process_ended   # "I'm wrapping up" — begin cleanup
```

The runner notices, waits `CLR_CLEANUP_SECONDS` (default `240`, i.e. 4 minutes) for the body to finish any cleanup, stops the loop, and exits.

The runner always waits the full `CLR_CLEANUP_SECONDS`, so size it to however long the desired cleanup window should be. Touch `process_ended` *before* you start cleaning up (or right as cleanup begins); when the window expires, the runner tears down the tmux session and exits.

The signal is consumed each cycle: the runner removes the `process_ended` file before launching the body. It's watched by the same backend as the trigger (`inotifywait` when available, otherwise an existence check every `CLR_POLL_SECONDS`).

> **Note:** `process_ended` uses `CLR_CLEANUP_SECONDS`, while the ordinary trigger and timer use `CLR_WAIT_SECONDS`.

### Watching it run

The session is named `looprunner-<your_script_name>`. Attach to watch live output:

```bash
tmux attach -t looprunner-do_work_sh
```

Detach (leaving everything running) with `Ctrl-b` then `d`.

### Stopping it

Either of these works:

- **process_ended file** — from the sandbox, touch the process_ended file:

  ```bash
  touch do_work.sh.process_ended
  ```

  The loop waits for the cleanup window, then kills the running program, tears down the tmux session, and exits.

- **Ctrl-C** — pressing `Ctrl-C` on the `cheeky-loop-runner` process (or `kill`ing it) triggers an immediate clean shutdown.

## How it works

Each cycle, `cheeky-loop-runner`:

1. Clears any stale `process_ended` file.
2. Sends `bash /abs/path/to/your_script.sh` into the tmux session and presses Enter.
3. Waits until **any** of: the `process_ended` file appears, the trigger file changes, or the run period elapses — whichever comes first. (These are watched via `inotifywait`, or an mtime/existence poll as a fallback.)
4. If `process_ended` fired, it waits `CLR_CLEANUP_SECONDS` (default 4 min) for the body to finish cleaning up, then kills the session and exits.
5. Otherwise (trigger or timer) it waits the configurable grace period (`CLR_WAIT_SECONDS`, default 4 min) so triggers coalesce and writes settle.
6. Sends four `Ctrl-C`s to the session (with a short gap between each) to terminate the program.
7. Pauses briefly so the program fully exits, then loops into the next cycle.

The session is created detached and rooted in your current directory (`tmux new-session -d -c "$PWD"`), and the script path is resolved to an absolute path up front, so changing the session's working directory never breaks the reference. Any leftover session with the same name is removed before a new run starts, so every run begins from a clean state.

## Configuration

The cycle length is the optional `run_minutes` argument (`0` = pure event-driven). Everything else is configurable via environment variables, so you don't have to edit the script:

| Variable | Default | Purpose |
| --- | --- | --- |
| `CLR_TRIGGER_FILE` | `<script>.trigger` in the sandbox | The fixed file your loop body touches to request an immediate (kill-and-restart) recycle. |
| `CLR_PROCESS_ENDED_FILE` | `<script>.process_ended` in the sandbox | The fixed file your loop body touches to request a cleanup wait followed by stopping the loop. |
| `CLR_WAIT_SECONDS` | `240` (4 min) | Grace/debounce wait before each trigger/timer restart, so rapid triggers coalesce and final writes settle. |
| `CLR_CLEANUP_SECONDS` | `240` (4 min) | Time the body is given to finish cleaning up after touching the `process_ended` file, before the runner exits. |
| `CLR_POLL_SECONDS` | `10` | Poll interval for fallback-watch. |
| `CLR_SETTLE_SECONDS` | `2` | Pause after the `Ctrl-C`s before restarting. |
| `CLR_WATCH_BACKEND` | `auto` | `auto`, `inotify`, or `poll`. `auto` uses `inotifywait` if present, else `poll`. |

Example — event-driven only, with a custom trigger file and a 5-second debounce:

```bash
CLR_TRIGGER_FILE=/tmp/recycle.flag CLR_WAIT_SECONDS=5 cheeky-loop-runner do_work.sh 0
```

## Notes and caveats

- The four `Ctrl-C`s are sent as terminal interrupts to whatever is running in the session. A program that ignores `SIGINT` won't be stopped by them — adjust the script if you need a different signal.
- The trigger file is matched by name in its directory. If you point `CLR_TRIGGER_FILE` at a path whose directory doesn't exist yet, create the directory first (the runner watches the directory, not a not-yet-existent file).
- With the `poll` backend, trigger detection happens within `CLR_POLL_SECONDS`; with `inotify` it's effectively instant. Either way, only modifications that occur after a run starts are counted.
- If your script finishes on its own before the period elapses and *doesn't* touch the `process_ended` file, the session simply sits at an idle shell prompt until the cycle ends (or a trigger fires); the `Ctrl-C`s then land harmlessly on that prompt. Touch `process_ended` to start the cleanup wait without waiting for the cycle timer.
- Running the same script name from two different sandboxes produces the same session name. If you need concurrent runs of identically named scripts, give them distinct names or adjust the `SESSION` variable.

## Contributing

Issues and pull requests are welcome. Keep it small and dependency-free — the whole point is a single portable Bash file.

## License

Released under the MIT License. Add a `LICENSE` file with your name and year, or swap in whichever license you prefer.
