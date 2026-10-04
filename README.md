# runbg

**Start a command on your server, log out, come back later and check on it.**

One small bash script. No dependencies, no setup beyond putting it on your PATH.

## The problem

You SSH into a server and need to run something long: a migration, a big
download, a backup. You could type:

```bash
nohup ./big-job.sh > /tmp/job.log 2>&1 &
```

...and then try to remember where the log went and what the PID was when you
come back two hours later. `runbg` does the same thing but in a form you won't
forget:

```bash
runbg ./big-job.sh
```

Then, from any later session (even after logging out and back in):

```bash
runbg log   # watch the output
runbg status  # is it still running?
runbg stop  # kill it
```

## Quick start

```bash
# 1. Put the script somewhere on your PATH
sudo cp runbg /usr/local/bin/runbg
sudo chmod +x /usr/local/bin/runbg

# 2. Try it
runbg sleep 30
runbg status   # -> RUNNING
runbg log      # -> shows the log, follows it while the job runs
runbg stop     # -> stops it

# 3. For real
runbg ./big-migration.sh
logout         # job keeps running
```

The job survives logout: it runs detached and immune to the hangup signal,
so closing your SSH session (or losing connection) does not kill it.

## Commands

| Command | What it does |
|---|---|
| `runbg <command> [args...]` | Start the command in the background. Output goes to `~/jobs/runbg.log`. |
| `runbg log` | Show the output so far. If the job is still running, keep following it (like `tail -f`) until you press Ctrl-C. |
| `runbg status` | RUNNING with PID, or FINISHED with exit code. |
| `runbg stop` | Stop the job: SIGTERM, wait up to 5 s, then SIGKILL. |
| `runbg -- <command...>` | Escape hatch for a command whose own name is `log`, `status`, or `stop`. |

Only one job per user at a time; starting a second one is refused until the
first finishes. That is a feature, not a bug: it keeps the state trivial and
`runbg log` always refers to *your* job.

## An example session

```console
$ runbg rsync -a /data/ backup@nas:/backup/
Started in background.
  PID:   12345
  Log:   /home/you/jobs/runbg.log
  Watch: runbg log

$ runbg status
RUNBG: RUNNING
PID: 12345

$ runbg log
...rsync output, following live...
^C

$ runbg stop
Stopped (exit code 143).
```

## Where things live

Everything is in one directory, `~/jobs` by default:

- `runbg.log`: the job's combined stdout and stderr
- `runbg.pid`: PID while the job is running (removed when it exits)
- `runbg.exit`: the exit code once it finishes

Set `RUNBG_DIR=/somewhere/else` to move the directory.

## Root jobs

Do not run `runbg sudo <command>`: the background job has no terminal, so
sudo cannot ask you for a password and the job will silently fail to start.
Run the whole thing as root instead:

```bash
sudo runbg <command>
```

Root's jobs live in `/root/jobs`, separate from yours.

## Why not nohup / systemd-run / tmux?

- `nohup cmd &` works, but you have to remember the redirect, find the PID
  later, and dig through `nohup.out`. runbg is the same idea with the
  bookkeeping done for you.
- `systemd-run` is excellent but needs systemd, and many servers, containers,
  and NAS boxes don't have it. runbg needs only bash and coreutils.
- `tmux` is overkill when all you want is "keep running, let me peek at the
  log later."

## Limitations

- One job per user at a time (by design, see above).
- `runbg stop` kills the job's process; if your command spawns its own
  background children, they may outlive the stop and need a manual kill.
- A recycled PID can rarely fool the running-check. If status looks wrong,
  check with `ps -p $(cat ~/jobs/runbg.pid)`.
- Each new job overwrites `runbg.log`. Copy it if you want to keep it.
- Not a supervisor: if the server reboots, the job is gone. For services that
  must survive reboots, use systemd.

## License

MIT, see [LICENSE](LICENSE).
