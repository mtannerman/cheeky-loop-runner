# cheeky-loop-runner

A tiny Bash utility that runs a shell script inside a [tmux](https://github.com/tmux/tmux) session on a timed **and event-driven** loop: it starts your script, lets it run until either a fixed period elapses (30 minutes by default) **or your script touches a trigger file**, kills it by sending four `Ctrl-C`s to the session, then restarts it — forever, until you stop it.

It's handy for jobs that you want to run in fixed-length bursts and restart cleanly: long-running scrapers, training or fuzzing runs, watchers, anything that should be cycled rather than left running indefinitely. With the trigger file, the job itself can also say "recycle me now" without waiting out the timer.

## Features

- **Run anything** — point it at any shell script; no execute bit or shebang required.
- **Event-driven restarts** — your script touches a fixed trigger file and the runner recycles it immediately, no waiting for the timer.
- **"I'm done" signalling** — when the body finishes on its own it can touch a `process_ended` file; the runner restarts the next cycle right away without bothering to send `Ctrl-C`.
- **Linux-native watching** — uses `inotifywait` when available (no polling), and falls back to a dependency-free mtime poll otherwise.
- **Sandbox-friendly** — the script runs in the directory you invoked `cheeky-loop-runner` from, so relative paths inside your script resolve where you expect.
- **Clean restarts** — each cycle sends four `Ctrl-C`s, then restarts from a fresh prompt.
- **Two ways to stop** — `Ctrl-C` the runner, or drop a stopfile (no need to find a PID).
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

Run in **pure event-driven mode** (no time ceiling — only the trigger file or stopfile recycle/stop it):

```bash
cheeky-loop-runner do_work.sh 0
```

On startup it prints the resolved script path, the working directory, the tmux session name, the cycle length, the trigger and stopfile paths, and the watch backend in use.

### Triggering a restart from your script (event-driven)

The runner watches a fixed **trigger file** — by default `<script-name>.trigger` in the sandbox (e.g. `do_work.sh.trigger`). Whenever that file is created, touched, or written, the runner recycles the loop body right away rather than waiting out the timer.

So from inside your loop body (or any other process), just touch it when you want a restart — for example after finishing a unit of work, noticing new input, or detecting a config change:

```bash
touch do_work.sh.trigger      # or: : >> do_work.sh.trigger
```

The runner waits a configurable grace period (`CLR_WAIT_SECONDS`, default 240s / 4 min) so a burst of touches coalesces into a single restart and any final writes settle, then sends the `Ctrl-C`s and restarts. Only changes that happen *after* a run begins count, so a stale trigger file left over from a previous run won't cause a spurious restart.

Under the hood this uses `inotifywait` when it's installed (instant, no polling) and otherwise falls back to checking the trigger file's modification time every `CLR_POLL_SECONDS`. Force one or the other with `CLR_WATCH_BACKEND=inotify|poll`.

### Signalling that the process is done (event-driven)

The trigger file means "I'm still running — kill me and start over." When instead your loop body **finishes its work and exits on its own**, it can say so by touching the **process_ended file** — by default `<script-name>.process_ended` in the sandbox (e.g. `do_work.sh.process_ended`):

```bash
touch do_work.sh.process_ended   # "I'm wrapping up" — then do cleanup and exit
```

The runner notices, skips the four `Ctrl-C`s entirely (there's nothing to forcibly interrupt), gives the body up to `CLR_CLEANUP_SECONDS` (default `240`, i.e. 4 minutes) to finish any cleanup and exit so the tmux pane returns to a shell prompt, then launches the next cycle.

This cleanup window is a ceiling, not a fixed delay only in spirit — the runner always waits the full `CLR_CLEANUP_SECONDS` before relaunching, so size it to however long your body's worst-case cleanup takes. Touch `process_ended` *before* you start cleaning up (or right as cleanup begins), and make sure the body has exited by the end of the window; otherwise the next command would be typed into the still-running process.

The signal is consumed each cycle: the runner removes the `process_ended` file before launching the body and again after acting on it, so a stale file left over from a previous run won't cause a spurious restart. It's watched by the same backend as the trigger (`inotifywait` when available, otherwise an existence check every `CLR_POLL_SECONDS`).

> **Note:** the runner does *not* kill the body on a `process_ended` signal, so the body must exit on its own within `CLR_CLEANUP_SECONDS`. If it's still running when the runner relaunches, the new command would be typed into the still-running process rather than the shell. Bump `CLR_CLEANUP_SECONDS` if your cleanup can take longer than 4 minutes.

### Watching it run

The session is named `looprunner-<your_script_name>`. Attach to watch live output:

```bash
tmux attach -t looprunner-do_work_sh
```

Detach (leaving everything running) with `Ctrl-b` then `d`.

### Stopping it

Either of these works:

- **Stopfile** — from the sandbox, create the stopfile named after your script:

  ```bash
  touch do_work.sh.stop
  ```

  The loop exits at the next check (within ~10 seconds), kills the running program, and tears down the tmux session. Remember to `rm do_work.sh.stop` before starting again, or the next run will exit immediately.

- **Ctrl-C** — pressing `Ctrl-C` on the `cheeky-loop-runner` process (or `kill`ing it) triggers the same clean shutdown.

## How it works

Each cycle, `cheeky-loop-runner`:

1. Checks for the stopfile and exits if it's present, then clears any stale `process_ended` file.
2. Sends `bash /abs/path/to/your_script.sh` into the tmux session and presses Enter.
3. Waits until **any** of: the `process_ended` file appears, the trigger file changes, the run period elapses, or the stopfile appears — whichever comes first. (These are watched via `inotifywait`, or an mtime/existence poll as a fallback; the stopfile is re-checked on every wake.)
4. If `process_ended` fired, it waits `CLR_CLEANUP_SECONDS` (default 4 min) for the body to finish cleaning up and exit, then skips straight to step 7 — no `Ctrl-C`. Otherwise (trigger or timer) it waits the configurable grace period (`CLR_WAIT_SECONDS`, default 4 min) so triggers coalesce and writes settle.
5. Sends four `Ctrl-C`s to the session (with a short gap between each) to terminate the program.
6. Pauses briefly so the program fully exits.
7. Loops into the next cycle.

The session is created detached and rooted in your current directory (`tmux new-session -d -c "$PWD"`), and the script path is resolved to an absolute path up front, so changing the session's working directory never breaks the reference. Any leftover session with the same name is removed before a new run starts, so every run begins from a clean state.

## Configuration

The cycle length is the optional `run_minutes` argument (`0` = pure event-driven). Everything else is configurable via environment variables, so you don't have to edit the script:

| Variable | Default | Purpose |
| --- | --- | --- |
| `CLR_TRIGGER_FILE` | `<script>.trigger` in the sandbox | The fixed file your loop body touches to request an immediate (kill-and-restart) recycle. |
| `CLR_PROCESS_ENDED_FILE` | `<script>.process_ended` in the sandbox | The fixed file your loop body touches to signal it finished on its own; the runner restarts without sending `Ctrl-C`. |
| `CLR_WAIT_SECONDS` | `240` (4 min) | Grace/debounce wait before each trigger/timer restart, so rapid triggers coalesce and final writes settle. |
| `CLR_CLEANUP_SECONDS` | `240` (4 min) | Time the body is given to finish cleaning up and exit after touching the `process_ended` file, before the next cycle starts. |
| `CLR_POLL_SECONDS` | `10` | Stopfile + fallback-watch poll interval. |
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
- If your script finishes on its own before the period elapses and *doesn't* touch the `process_ended` file, the session simply sits at an idle shell prompt until the cycle ends (or a trigger fires); the `Ctrl-C`s then land harmlessly on that prompt. Touch `process_ended` to skip that wait and recycle immediately.
- Running the same script name from two different sandboxes produces the same session name. If you need concurrent runs of identically named scripts, give them distinct names or adjust the `SESSION` variable.

## Contributing

Issues and pull requests are welcome. Keep it small and dependency-free — the whole point is a single portable Bash file.

## License

Released under the MIT License. Add a `LICENSE` file with your name and year, or swap in whichever license you prefer.
