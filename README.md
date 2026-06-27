# cheeky-loop-runner

A tiny Bash utility that runs a shell script inside a [tmux](https://github.com/tmux/tmux) session on a timed loop: it starts your script, lets it run for a fixed period (30 minutes by default), kills it by sending four `Ctrl-C`s to the session, then restarts it — forever, until you stop it.

It's handy for jobs that you want to run in fixed-length bursts and restart cleanly: long-running scrapers, training or fuzzing runs, watchers, anything that should be cycled rather than left running indefinitely.

## Features

- **Run anything** — point it at any shell script; no execute bit or shebang required.
- **Sandbox-friendly** — the script runs in the directory you invoked `cheeky-loop-runner` from, so relative paths inside your script resolve where you expect.
- **Clean restarts** — each cycle sends four `Ctrl-C`s, then restarts from a fresh prompt.
- **Two ways to stop** — `Ctrl-C` the runner, or drop a stopfile (no need to find a PID).
- **Predictable sessions** — the tmux session is named after your script, so you can attach and watch it live.
- **Configurable period** — default 30-minute cycles, overridable per run.

## Requirements

- Bash 4+
- `tmux`

On Debian/Ubuntu: `sudo apt install tmux`. On macOS: `brew install tmux`.

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

On startup it prints the resolved script path, the working directory, the tmux session name, the cycle length, and the exact stopfile path to use.

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

1. Checks for the stopfile and exits if it's present.
2. Sends `bash /abs/path/to/your_script.sh` into the tmux session and presses Enter.
3. Waits the configured period, polling for the stopfile every 10 seconds.
4. Sends four `Ctrl-C`s to the session (with a short gap between each) to terminate the program.
5. Pauses briefly so the program fully exits, then loops.

The session is created detached and rooted in your current directory (`tmux new-session -d -c "$PWD"`), and the script path is resolved to an absolute path up front, so changing the session's working directory never breaks the reference. Any leftover session with the same name is removed before a new run starts, so every run begins from a clean state.

## Configuration

Most knobs are exposed as the optional `run_minutes` argument and the stopfile convention. A few additional defaults live in the configuration block near the top of the script and can be edited directly:

- `SETTLE_SECONDS` — pause after the `Ctrl-C`s before restarting (default `2`).
- `POLL_SECONDS` — how often the stopfile is checked while waiting (default `10`).

## Notes and caveats

- The four `Ctrl-C`s are sent as terminal interrupts to whatever is running in the session. A program that ignores `SIGINT` won't be stopped by them — adjust the script if you need a different signal.
- If your script finishes on its own before the period elapses, the session simply sits at an idle shell prompt until the cycle ends; the `Ctrl-C`s then land harmlessly on that prompt.
- Running the same script name from two different sandboxes produces the same session name. If you need concurrent runs of identically named scripts, give them distinct names or adjust the `SESSION` variable.

## Contributing

Issues and pull requests are welcome. Keep it small and dependency-free — the whole point is a single portable Bash file.

## License

Released under the MIT License. Add a `LICENSE` file with your name and year, or swap in whichever license you prefer.
