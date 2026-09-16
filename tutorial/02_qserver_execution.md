# Queue Server Execution

## Overview

In this step, you'll learn to run your BITS instrument through the **bluesky
queueserver** (QS). Instead of typing plans into an interactive IPython session,
you submit them to a queue that a long-running host process executes on your
behalf. This is how BITS instruments are operated remotely, unattended, and by
multiple clients at once.

You'll start and manage the **QS host** with the `qs_host.sh` script, watch and
drive the queue with the **`queue-monitor`** GUI, and submit plans from Python
with the queueserver API.

**Time**: ~30 minutes
**Goal**: Start the queueserver host, connect the monitor, and run a plan from the queue

## Prerequisites

✅ Completed Step 01 (IPython Interactive Execution)
✅ Working instrument package (`from my_instrument.startup import *` succeeds)
✅ Working device configurations and at least one plan to run
✅ IOCs running and responsive
✅ **Redis** available on `localhost:6379` (the QS host requires it)

## Understanding the Queue Server

### Interactive IPython vs. Queue Server

| Aspect | IPython (`RE(...)`) | Queue Server |
|--------|---------------------|--------------|
| Who runs the plan | You, in your terminal | A separate host process |
| Concurrency | One session | Many clients, one shared queue |
| Remote control | No | Yes (over ZMQ) |
| Unattended batches | Manual | Queue executes in order |
| Same `startup.py`? | Yes | Yes (loaded by the host) |
| Plan style | `RE(bp.count([det]))` | Submit `BPlan("count", [det])` |

The key idea: the queueserver host loads the **same `startup.py`** as your
IPython session, so the same devices and plans are available. You just talk to it
through a queue instead of calling `RE()` directly.

### The pieces

```
                submit plans                 executes
   client  ─────────────────►   QS host   ─────────────►  RunEngine + devices
 (queue-monitor,              (start-re-manager,           (your startup.py)
  Python API,                  runs your startup.py)
  qserver CLI)
        ▲                          │
        └────── status / console ──┘
                   (ZMQ)
```

- **QS host** — the `start-re-manager` process. It owns the RunEngine and your
  devices. Managed by `qs_host.sh`.
- **Redis** — stores the queue and history so they survive restarts.
- **Clients** — anything that connects over ZMQ: the `queue-monitor` GUI, the
  `qserver` command-line client, or the Python `REManagerAPI`.

### Where the files live

Everything for the queueserver lives in your instrument's `qserver/` directory
and a management script under `scripts/`:

```
my_instrument/
├── scripts/
│   └── my_instrument_qs_host.sh     # start/stop/status the QS host
└── src/my_instrument/
    ├── startup.py                   # loaded by the QS host
    └── qserver/
        ├── qs-config.yml            # host configuration (network, startup, worker)
        └── user_group_permissions.yaml  # which plans/devices each group may use
```

> **Note**: The `qserver/` directory and the `*_qs_host.sh` script are named
> after **your** instrument. This tutorial uses `my_instrument`; substitute your
> own instrument name throughout.

## Starting the Queue Server Host


### 1. Look at the host management script

The `qs_host.sh` script wraps the QS host in a [GNU `screen`](https://www.gnu.org/software/screen/manual/screen.html)
session so it keeps running after you disconnect. Ask it for help:

```bash
cd my_instrument   # the instrument working directory
./scripts/my_instrument_qs_host.sh help
```

**Expected output:**

```
Usage: my_instrument_qs_host.sh {start|stop|restart|status|checkup|console|run} [NAME]

    COMMANDS
        console   attach to process console if process is running in screen
        checkup   check that process is running, restart if not
        restart   restart process
        run       run process in console (not screen)
        start     start process
        status    report if process is running
        stop      stop process

    OPTIONAL TERMS
        NAME      name of process (default: bluesky_queueserver-default)
```

### 2. Start (or restart) the host

`restart` is the usual way to bring the host up: it stops any existing host, then
starts a fresh one. This is safe to run whether or not one is already running.

```bash
# Make sure your environment is active first
conda activate bits_env

# (Re)start the QS host in a screen session
./scripts/my_instrument_qs_host.sh restart
```

**Expected output:**

```
Starting bluesky_queueserver-default
```

### 3. Check the status

```bash
./scripts/my_instrument_qs_host.sh status
```

**Expected output (running):**

```
12345.bluesky_queueserver-default is running (pid=12345) in a screen session (pid=12340)
```

### 4. Watch the host console (optional)

To see the host's live log output (device creation, plan execution, errors),
attach to its screen session. Detach again with **`Ctrl-a` then `d`** — do *not*
press `Ctrl-c`, which would stop the host.

```bash
./scripts/my_instrument_qs_host.sh console
```

<details>
<summary>What each command does:</summary>

command | description
--- | ---
`start` | start the host if it isn't already running
`stop` | stop the running host
`restart` | stop then start — the usual (re)start command
`status` | report whether the host is running, with its PID
`checkup` | restart the host only if it is not running (good for cron)
`console` | attach to the host's screen session to watch its output
`run` | run the host in the foreground (diagnostics only — no screen)

</details>

### Common startup errors

```
PROCESS 'start-re-manager': file not found. CONDA_PREFIX='...'
```
→ Your environment isn't active, or `bluesky-queueserver` isn't installed.
Run `conda activate bits_env` and retry.

```
Must manage queueserver process on <host>.  This is <other-host>.
```
→ The script pins the host to one machine (`QS_HOSTNAME`). Run it on that host,
or edit `QS_HOSTNAME` in the script.

## Understanding `qs-config.yml`

The host reads all of its configuration from `qserver/qs-config.yml`. You rarely
need to change it, but you should recognize its sections:

```yaml
network:
    redis_addr: localhost:6379          # where the queue is stored
    zmq_control_addr: tcp://*:60615     # clients submit/control here
    zmq_info_addr: tcp://*:60625        # clients read console output here
    zmq_publish_console: true

operation:
    console_logging_level: NORMAL       # SILENT | QUIET | NORMAL | VERBOSE
    print_console_output: true
    update_existing_plans_and_devices: ENVIRONMENT_OPEN
    user_group_permissions_reload: ON_STARTUP

startup:
    keep_re: true
    startup_module: my_instrument.startup   # <-- your instrument's startup.py
    existing_plans_and_devices_path: ./
    user_group_permissions_path: ./

worker:
    use_ipython_kernel: true
    ipython_matplotlib: qt5
```

Key fields:

| Field | Meaning |
|-------|---------|
| `redis_addr` | Redis host:port for the queue/history |
| `zmq_control_addr` | Port clients use to submit and control plans |
| `startup_module` | The Python module the host imports to build devices/plans — this must be **your** instrument's `startup.py` |
| `user_group_permissions_path` | Directory holding `user_group_permissions.yaml` |

> **`bits-create` wires this for you.** When you scaffold an instrument, the
> `startup_module` is set to `<your_instrument>.startup` automatically. If you
> renamed the package, check this value.

## Running Plans in the Queue Server

Because the host loads your `startup.py`, the plans available in the queue are the
same ones you use in IPython — but you refer to them by **name** (a string), not
by calling them. For example, IPython's `RE(bp.count([scaler1], num=5))` becomes
the queue item `BPlan("count", ["scaler1"], num=5)`.

### Option A — The `queue-monitor` GUI (recommended to start)

The bluesky queueserver ships a Qt GUI for building and running the queue. Launch
it in the background:

```bash
queue-monitor &
```

Typical workflow in the GUI:

1. **Connect** — the monitor connects to the host over ZMQ automatically (it uses
   the same default addresses as `qs-config.yml`).
2. **Open the environment** — click **"Open"** (environment). This tells the host
   to import your `startup.py` and create the devices. Watch the console pane for
   device-creation messages.
3. **Add a plan** — pick a plan (e.g. `count`), fill in the arguments (detectors,
   `num`, `delay`), and add it to the queue.
4. **Run the queue** — click **"Start"**. The host executes queued items in order;
   the console and plot panes show live progress.
5. **Control** — pause / resume / stop / clear the queue, and inspect the history
   of completed runs.

<details>
<summary>Tips for the GUI:</summary>

- The plans and devices shown are exactly those permitted by
  `user_group_permissions.yaml` for your group — if a plan is missing, check
  permissions (see below).
- "Open environment" is separate from "Start queue". You open the environment
  once, then start/stop the queue as many times as you like.
- Closing the environment frees the devices; reopening re-runs `startup.py`.

</details>

### Option B — The Python API

You can drive the queue programmatically. This is the basis for automation and
scripted batches.

```python
# Connect to the Queue Server over ZMQ
from bluesky_queueserver_api import BPlan
from bluesky_queueserver_api.zmq import REManagerAPI

api = REManagerAPI()

# 1. Open the environment (loads your startup.py on the host)
api.environment_open()
api.wait_for_idle()          # wait until the environment is ready

# 2. Build a plan by NAME and add it to the queue
#    Equivalent to RE(bp.count([scaler1], num=5)) in IPython
plan = BPlan("count", ["scaler1"], num=5)
api.item_add(plan)

# 3. Start the queue
api.queue_start()

# 4. Check status
print(api.status())
```

<details>
<summary>More client operations:</summary>

```python
# Add several items, then run them all in order
api.item_add(BPlan("scan", ["scaler1"], "m1", -1, 1, 11))
api.item_add(BPlan("rel_scan", ["scaler1"], "m1", -0.5, 0.5, 11))

# Inspect the queue and history
print(api.queue_get())        # pending items
print(api.history_get())      # completed items

# Control execution
api.queue_stop()              # stop after the current item finishes
api.re_pause()                # pause the running plan
api.re_resume()               # resume it

# Clean up when finished
api.environment_close()
```

**Note**: device and detector names are passed as **strings** (`"scaler1"`,
`"m1"`) — the host resolves them to the objects created by your `startup.py`.

</details>

### Option C — The `qserver` command-line client

The `qserver` CLI (from `bluesky-queueserver`) is handy for quick checks and
shell scripts:

```bash
qserver status                       # host + queue status
qserver environment open             # load startup.py on the host
qserver queue add plan '{"name": "count", "args": [["scaler1"], 5]}'
qserver queue start                  # run the queue
qserver environment close
```

## Permissions: which plans and devices are allowed

The host reads `qserver/user_group_permissions.yaml` to decide what each user
group may run. Groups like `root` and `primary` typically allow everything, while
a restricted group (e.g. `test_user`) allows only certain plans/devices via
regular-expression patterns:

```yaml
user_groups:
  primary:                     # beamline staff: all plans and devices
    allowed_plans:
      - ":.*"                  # allow all plans
    allowed_devices:
      - ":?.*:depth=5"         # allow all devices and subdevices
  test_user:                   # limited access
    allowed_plans:
      - ":^count"              # only plans starting with "count"
      - ":scan$"               # ...and ending with "scan"
    allowed_devices:
      - ":^det:?.*"
      - ":^motor:?.*"
```

If a plan or device you expect is missing from the monitor, this file is the
first place to check. The host reloads it on startup (`user_group_permissions_reload: ON_STARTUP`),
so restart the host after editing:

```bash
./scripts/my_instrument_qs_host.sh restart
```

## Verifying an End-to-End Run

Put it all together to confirm your queueserver works:

```
1. Redis reachable      → verify: `redis-cli ping` returns PONG
2. Host running         → verify: `./scripts/my_instrument_qs_host.sh status` shows "is running"
3. Environment open     → verify: monitor shows devices, or `qserver status` shows environment open
4. Plan queued          → verify: `count` appears in the queue
5. Queue executed       → verify: run appears in history with "success"
```

```python
# Scripted version of the check
from bluesky_queueserver_api import BPlan
from bluesky_queueserver_api.zmq import REManagerAPI

api = REManagerAPI()
api.environment_open(); api.wait_for_idle()

api.item_add(BPlan("count", ["scaler1"], num=1))
api.queue_start()
api.wait_for_idle()

history = api.history_get()
last = history["items"][-1]
print(f"Last run exit status: {last['result']['exit_status']}")   # expect: completed
```

## Troubleshooting

### Host won't start

```bash
# Is it already running?
./scripts/my_instrument_qs_host.sh status

# Watch the console for the real error (startup.py import failures show here)
./scripts/my_instrument_qs_host.sh console
```

- **Import error in the console** → your `startup.py` fails to import. Reproduce
  it interactively first: `ipython -c "from my_instrument.startup import *"`.
- **`start-re-manager: file not found`** → environment not active / package not
  installed.
- **Redis connection refused** → start Redis, confirm `redis-cli ping`.

### Client can't connect

- Confirm the host is running (`status`).
- Confirm you're pointing at the same `zmq_control_addr` as `qs-config.yml`
  (default `tcp://localhost:60615`).
- Firewalls can block ports `60615`/`60625` for remote clients.

### Plan rejected or missing

- Not in `user_group_permissions.yaml` for your group → add it and restart.
- Device/plan names must be **strings** that exist in your `startup.py`.
- Open the environment before adding/running plans.

### Stuck queue or plan

```python
api.re_pause()       # pause the running plan
api.re_stop()        # stop the current plan
api.queue_stop()     # stop the queue after the current item
```

If the host is unrecoverable, restart it:

```bash
./scripts/my_instrument_qs_host.sh restart
```

## Best Practices

1. **Start clean**: `restart` the host after changing `startup.py`, device
   configs, plans, or permissions — the host caches them on import.
2. **Watch the console** during first runs (`console`) to catch import and
   connection issues early. Detach with `Ctrl-a d`, never `Ctrl-c`.
3. **Test plans in IPython first**: if `RE(bp.count([scaler1]))` works
   interactively, `BPlan("count", ["scaler1"])` will work in the queue.
4. **Keep the queue readable**: add descriptive metadata and clear old history
   periodically.
5. **Use `checkup` for resilience**: a cron job running
   `./scripts/my_instrument_qs_host.sh checkup` restarts the host if it dies.

## Deliverables

After completing this step, you should be able to:

- ✅ Start, stop, and check the QS host with `qs_host.sh`
- ✅ Explain the roles of the host, Redis, and clients
- ✅ Read the key fields of `qs-config.yml`
- ✅ Open the environment and run a plan from `queue-monitor`
- ✅ Submit and control plans from the Python `REManagerAPI`
- ✅ Adjust and reload `user_group_permissions.yaml`
- ✅ Troubleshoot common host and client problems

## Next Steps

With remote/queued operation working, continue building out your instrument:

- **Next Step**: [Device Configuration](03_device_configuration.md) — connect real
  IOC devices that the queueserver will drive.
- Then: [Plan Development](04_plan_development.md) — write custom plans, which
  become available in the queue automatically once loaded by `startup.py`.

---

## Reference: `qs_host.sh` Commands

| Command | Purpose |
|---------|---------|
| `./scripts/my_instrument_qs_host.sh start` | Start the host |
| `./scripts/my_instrument_qs_host.sh stop` | Stop the host |
| `./scripts/my_instrument_qs_host.sh restart` | Stop then start (usual command) |
| `./scripts/my_instrument_qs_host.sh status` | Report running state + PID |
| `./scripts/my_instrument_qs_host.sh checkup` | Restart only if not running |
| `./scripts/my_instrument_qs_host.sh console` | Attach to the host's screen session |
| `./scripts/my_instrument_qs_host.sh run` | Run in foreground (diagnostics) |

## Reference: Client Quick Commands

| Task | GUI | Python API | CLI |
|------|-----|------------|-----|
| Open environment | "Open" button | `api.environment_open()` | `qserver environment open` |
| Add a plan | Add-to-queue form | `api.item_add(BPlan(...))` | `qserver queue add plan '{...}'` |
| Start queue | "Start" button | `api.queue_start()` | `qserver queue start` |
| Status | status panels | `api.status()` | `qserver status` |
| Pause / resume | pause / resume | `api.re_pause()` / `api.re_resume()` | `qserver re pause` / `qserver re resume` |
| Close environment | "Close" button | `api.environment_close()` | `qserver environment close` |

## Reference: IPython → Queue Server Translation

| IPython (interactive) | Queue Server (`BPlan`) |
|-----------------------|------------------------|
| `RE(bp.count([scaler1], num=5))` | `BPlan("count", ["scaler1"], num=5)` |
| `RE(bp.scan([scaler1], m1, -1, 1, 11))` | `BPlan("scan", ["scaler1"], "m1", -1, 1, 11)` |
| `RE(bp.rel_scan([scaler1], m1, -0.5, 0.5, 11))` | `BPlan("rel_scan", ["scaler1"], "m1", -0.5, 0.5, 11)` |
| `RE(my_custom_plan(m1, scaler1))` | `BPlan("my_custom_plan", "m1", "scaler1")` |

Custom plans loaded by `startup.py` are available in the queue automatically —
refer to them by their function name as a string.
