# IPython Interactive Execution

## Overview

In this step, you'll learn to use your BITS instrument interactively with IPython. This is the primary way scientists interact with Bluesky for real-time data acquisition, analysis, and troubleshooting.

**Time**: ~30 minutes  
**Goal**: Master interactive operation of your instrument

## Prerequisites

✅ Completed Steps 0-3  
✅ BITS-Starter repository set up  
✅ Working device configurations  
✅ Custom plans created and tested  
✅ IOCs running and responsive

## Understanding IPython for Bluesky

### Why IPython?

IPython provides enhanced interactive Python with:
- **Magic commands**: `%wa`, `%ct`, `%mov` for device control
- **Tab completion**: Discover available methods and attributes
- **Command history**: Recall and modify previous commands
- **Rich display**: Better formatting of results
- **Integration**: Seamless with Bluesky and BITS

### IPython vs Standard Python

| Feature | Standard Python | IPython |
|---------|----------------|---------|
| Device listing | `print(devices)` | `%wa` |
| Help system | `help(function)` | `function?` |
| Command recall | Limited | Full history with search |
| Tab completion | Basic | Enhanced with context |
| Magic commands | None | Many Bluesky-specific commands |

## Starting Your Interactive Session

### 1. Launch IPython

```bash
# Ensure your environment is active
conda activate bits_env

# Start IPython
ipython
```

### 2. Load Your Instrument

```python
# Load everything from your instrument
from my_instrument.startup import *

# This imports:
# - RunEngine (RE)
# - All configured devices
# - Standard Bluesky plans (bp.*)
# - Plan stubs (bps.*)
# - Your custom plans
# - Data catalog (cat)
```

**Expected output:**
```
INFO:...:Loading configuration from .../iconfig.yml
INFO:...:Creating devices from devices.yml
INFO:...:RunEngine initialized
INFO:...:Data catalog 'temp' ready
INFO:...:Custom plans loaded
```

### 3. Verify Everything Loaded

```python
# Check RunEngine status
print(f"RunEngine state: {RE.state}")
print(f"Data catalog: {cat.name}")

# List available devices
%wa

# Check recent runs
list(cat)[-3:]  # Last 3 runs (if any exist)
```

## Essential IPython Magic Commands

### Device Management Commands

```python
# List all devices with current values
%wa

# List specific device types
%wa motors     # Only motors
%wa detectors  # Only detectors
%wa baseline   # Only baseline devices

# Move motors
%mov sim_motor 1.5   # Move sim_motor to (absolute) position 1.5
%movr sim_motor 0.1  # Move sim_motor (relative) by +0.1

# Count (labeled devices)
%ct                  # Count all devices with the "detectors" label (here: sim_det)
%ct baseline         # Count devices with "baseline" label
```


### Read

command | description
--- | ---
`OBJECT.get()` | low-level command to show value of ophyd *Signal* named `OBJECT`
`OBJECT.read()` | data acquisition command, includes timestamp
`listdevice(OBJECT)` | table-version of `.read()`
`MOTOR.position` | get readback, only for motor objects
`MOTOR.readback.get()` | alternative to `MOTOR.position` (simulated `SynAxis` motor)
`OBJECT.summary()` | more information about `OBJECT`

<details>
<summary>Examples:</summary>

<pre>
In [10]: <b>sim_motor.setpoint.get()</b>
Out[10]: 0

In [11]: <b>sim_motor.setpoint.read()</b>
Out[11]: {'motor_setpoint': {'value': 0, 'timestamp': 1789672950.539895}}

In [12]: <b>listdevice(sim_motor)</b>
================== ===== ==========================
data name          value timestamp
================== ===== ==========================
motor              0.0   2026-09-17 14:25:36.229619
motor_setpoint     0.0   2026-09-17 14:25:36.229487
motor_velocity     1     2026-09-17 14:25:22.751998
motor_acceleration 1     2026-09-17 14:25:22.752005
motor_unused       1     2026-09-17 14:25:22.752010
================== ===== ==========================

Out[12]: <pyRestTable.rest_table.Table at 0x7fe0649cbd00>

In [13]: <b>sim_motor.position</b>
Out[13]: 0

In [14]: <b>sim_motor.summary()</b>
data keys (* hints)
-------------------
*motor
 motor_setpoint

read attrs
----------
readback             _ReadbackSignal     ('motor')
setpoint             _SetpointSignal     ('motor_setpoint')

config keys
-----------
motor_acceleration
motor_velocity

configuration attrs
-------------------
velocity             Signal              ('motor_velocity')
acceleration         Signal              ('motor_acceleration')

unused attrs
------------
unused               Signal              ('motor_unused')
</pre>

</details>

## Move

command | description
--- | ---
`%mov MOTOR value` | interactive command move MOTOR to value (command line only)
`%movr MOTOR value` | interactive command relative move (command line only)
`MOTOR.set(value)` | ophyd command to `%mov` (simulated `SynAxis` uses `.set()`, not `.move()`)
`MOTOR.setpoint.put(value)` | ophyd to set the motor setpoint but not wait
`bps.mv(MOTOR, value)` | bluesky plan command to move and wait for completion
`bps.mv(MOTOR.setpoint, value)` | bluesky plan command, same as above
`bps.mvr(MOTOR, value)` | bluesky plan command, relative move

<details>
<summary>Examples:</summary>

<pre>
In [15]: <b>%mov sim_motor 1</b>

In [16]: <b>%movr sim_motor -1</b>

In [17]: <b>sim_motor.set(.5)</b>
Out[17]: MoveStatus(done=True, pos=sim_motor, elapsed=0.0, success=True, settle_time=0.0)

In [18]: <b>sim_motor.setpoint.put(1)</b>

In [19]: <b>RE(bps.mv(sim_motor, 0))</b>
Out[19]: ()
</pre>

</details>


## Count

command | description
--- | ---
`%ct` | count _all_ objects with label `detectors` and format output (command line only)
`sim_det.trigger().wait(); sim_det.read()` | ophyd command to count `sim_det`
`bp.count([sim_det])` | bluesky plan to count

The simulated `sim_det` (an `ophyd.sim.SynGauss`) has no count time. Instead of a
count time, it computes a Gaussian value from `sim_motor`'s position. You shape that
simulated signal with these configuration signals:

signal | meaning
--- | ---
`sim_det.Imax` | peak intensity
`sim_det.center` | motor position of the peak center
`sim_det.sigma` | peak width
`sim_det.noise` | `none`, `poisson`, or `uniform`

<details>
<summary>Examples:</summary>

<pre>
In [20]: <b>sim_det.Imax.get()</b>
Out[20]: 1

In [21]: <b>RE(bps.mv(sim_det.Imax, 10000))</b>
Out[21]: ()

In [22]: <b>sim_det.Imax.get()</b>
Out[22]: 10000

In [23]: <b>%ct</b>
[This data will not be saved. Use the RunEngine to collect data.]
noisy_det                      9811.780994809134

In [24]: <b>sim_det.trigger().wait()</b>

In [25]: <b>sim_det.read()</b>
Out[25]:
OrderedDict([('noisy_det', {'value': 9805.12, 'timestamp': 1789672950.5404348})])

In [26]: <b>sim_det.trigger().wait(); sim_det.read()</b>
Out[26]:
OrderedDict([('noisy_det', {'value': 9817.44, 'timestamp': 1789672950.5404348})])
</pre>

</details>

## List, Describe, Summary

command | description
--- | ---
`wa` | show all labeled objects
`listobjects()` | table of all global objects
`listruns()` | table of runs (default: last 20)
`OBJECT.describe()` | OBJECT metadata: PV, type, units, limits, precision, ... (written as part of a run)
`OBJECT.summary()` | OBJECT details in human readable terms

<details>
<summary>Examples:</summary>

<pre>
In [43]: <b>%wa</b>
detectors
  Local variable name                    Ophyd name (to be recorded as metadata)
  sim_det                                sim_det

motors
  Positioner                     Value       Low Limit   High Limit  Offset
  sim_motor                      0           AttributeError AttributeError AttributeError

  Local variable name                    Ophyd name (to be recorded as metadata)
  sim_motor                              sim_motor


In [44]: <b>listobjects()</b>
========= =============== ============== =========
name      class           PV (or prefix) label(s)
========= =============== ============== =========
sim_det   SynGauss                       detectors
sim_motor SynAxis                        motors
========= =============== ============== =========

Out[44]: <pyRestTable.rest_table.Table at 0x7fe064171fd0>

In [45]: <b>listruns()</b>
======= =================== ========= ===========
scan_id time                plan_name detectors
======= =================== ========= ===========
7       2026-09-17 14:25:36 count     ['sim_det']
6       2026-09-17 14:25:36 scan      ['sim_det']
======= =================== ========= ===========

Out[45]: <pyRestTable.rest_table.Table at 0x7fe064174190>

In [48]: <b>sim_det.describe()</b>
Out[48]:
OrderedDict([('noisy_det',
              {'source': 'SIM:noisy_det',
               'dtype': 'number',
               'shape': [],
               'precision': 3})])

In [49]: <b>sim_det.summary()</b>
data keys (* hints)
-------------------
*noisy_det

read attrs
----------
val                  SynSignal           ('noisy_det')

config keys
-----------
noisy_det_Imax
noisy_det_center
noisy_det_noise
noisy_det_noise_multiplier
noisy_det_sigma

configuration attrs
-------------------
Imax                 Signal              ('noisy_det_Imax')
center               Signal              ('noisy_det_center')
sigma                Signal              ('noisy_det_sigma')
noise                EnumSignal          ('noisy_det_noise')
noise_multiplier     Signal              ('noisy_det_noise_multiplier')

unused attrs
------------

</pre>

</details>

> **Note:** In the `%wa` output above, the `Low Limit`, `High Limit`, and `Offset`
> columns show `AttributeError` because the simulated `sim_motor` (`ophyd.sim.SynAxis`)
> has no soft limits — unlike an `EpicsMotor`. This is expected for simulated devices.

## Bluesky Plans _vs_. Command-line Actions

There is a difference in the commands to use depending on the context.

context | blocking? | command style
--- | --- | ---
plan function | NOT allowed | call Bluesky [plans](https://blueskyproject.io/bluesky/plans.html) written as [generator](https://wiki.python.org/moin/Generators) functions using `yield from a_plan()`
command line | allowed | use magics (such as `%mov`), `.put()`, and/or `RE(a_plan())`

<details>
<summary>Examples:</summary>

<b>plan function</b>

Write a plan to move `sim_motor` to a position:

```py
def move_sample(position):
    """
    plan: move sim_motor to a position, then let it settle.
    """
    yield from bps.mv(sim_motor, position)
    yield from bps.sleep(0.5)       # allow the stage to settle
```

Then, call `move_sample()` from another plan such as

```py
    yield from move_sample(0)
```

<b>command line actions</b>

There are (at least) three different ways to move `sim_motor` from the command
line:

```py
# use bluesky Magic command
%mov sim_motor 1.5

# or use the ophyd object
sim_motor.set(1.5)

# or use the bluesky RunEngine
RE(bps.mv(sim_motor, 1.5))
```

NOTE: On the command line, we can ignore the 0.5 s sleep needed by automated
procedures.

</details>


### RunEngine Commands

```python
# RunEngine control
RE.state             # Current state
RE.pause()           # Pause current scan
RE.resume()          # Resume paused scan
RE.stop()            # Stop current scan (abrupt)
RE.abort()           # Return RunEngine to idle gracefully

# Scan history
RE.md                # Current metadata
RE.md['scan_id']     # Current scan ID
```

## Interactive Device Testing

Open your IPython terminal and try the following

### 1. Motor Testing

```python
# Check motor status
print(f"Motor sim_motor position: {sim_motor.position}")
print(f"Motor connected: {sim_motor.connected}")

# Test small moves
initial_pos = sim_motor.position
print(f"Starting at: {initial_pos=}")

# Move relative
RE(bps.mvr(sim_motor, 0.1))
print(f"After +0.1: {sim_motor.position=}")

# Move back
RE(bps.mvr(sim_motor, -0.1))
print(f"Back to: {sim_motor.position=}")

# Absolute move
RE(bps.mv(sim_motor, 0.0))
print(f"At zero: {sim_motor.position=}")
```

### 2. Detector Testing

```python
# Test detector reading
print(f"Detector connected: {sim_det.connected}")
print(f"Current reading: {sim_det.read()}")

# Simple count
RE(bp.count([sim_det], num=1, delay=1))

# Count several times
RE(bp.count([sim_det], num=3, delay=2))

# Shape the simulated signal (sim_det has no count time)
RE(bps.mv(sim_det.Imax, 10000))   # peak intensity
RE(bps.mv(sim_det.center, 0))     # peak center (sim_motor position)
RE(bps.mv(sim_det.sigma, 1))      # peak width
RE(bps.mv(sim_det.noise, "uniform"))  # none, poisson, or uniform
```

### 3. Combined Device Testing

```python
# Test motor and detector together
print("Testing coordinated motion and detection...")

# Move motor and count at each position
positions = [-0.5, 0, 0.5]
for pos in positions:
    print(f"\nMoving to {pos}")
    RE(bps.mv(sim_motor, pos))
    
    print(f"Counting at {sim_motor.position=}")
    RE(bp.count([sim_det], num=1))
```

## Interactive Scanning

### 1. Basic Scans

```python
# Simple scan
RE(bp.scan([sim_det], sim_motor, -1, 1, 11))

# Relative scan (around current position)
RE(bp.rel_scan([sim_det], sim_motor, -0.5, 0.5, 11))

# List scan (specific positions)
positions = [-1, -0.25, 0, 0.35, 1]  # note irregular spacing
RE(bp.list_scan([sim_det], sim_motor, positions))

# Count without motion
RE(bp.count([sim_det], num=5, delay=1))
```

### 2. Multi-dimensional Scans

Grid scans move more than one motor. This training instrument defines a single
simulated motor (`sim_motor`), so a 2-D grid scan needs a second motor added to
`devices.yml` (for example another `ophyd.sim.motor`). Once a second motor
(`sim_motor2`) exists, a grid scan looks like:

```python
# Grid scan with 2 motors (requires a second motor, e.g. sim_motor2)
RE(bp.grid_scan([sim_det],
                sim_motor, -1, 1, 5,     # 5 points from -1 to 1
                sim_motor2, -0.5, 0.5, 3, # 3 points from -0.5 to 0.5
                snake_axes=[sim_motor2])) # Snake pattern in sim_motor2
```

### 3. Using Your Instrument's Example Plans

The BITS instrument ships example plans in `plans/sim_plans.py` that drive
`sim_motor` and `sim_det` directly:

```python
# Print sim_motor position and sim_det reading (no data saved)
RE(sim_print_plan())

# Count sim_det a few times
RE(sim_count_plan(num=3))

# Relative scan of sim_det vs sim_motor, shaping the simulated peak
RE(sim_rel_scan_plan(num=11, imax=10000, center=0, sigma=1, noise="uniform"))
```

## Modifying your instrument
### Your iconfig file

Go inside your iconfig.yaml file, located in the configs folder of your instrument. Play around with the values and see what happens to your ipython session
```yaml
# Configuration for the Bluesky instrument package.

# identify the version of this iconfig.yml file
ICONFIG_VERSION: 2.0.1

# Add additional configuration for use with your instrument.

### The short name for the databroker catalog.
DATABROKER_CATALOG: &databroker_catalog temp

### RunEngine configuration
RUN_ENGINE:
    DEFAULT_METADATA:
        beamline_id: demo_instrument
        instrument_name: Most Glorious Scientific Instrument
        proposal_id: commissioning
        databroker_catalog: *databroker_catalog

    ### EPICS PV to use for the `scan_id`.
    ### Default: `RE.md["scan_id"]` (not using an EPICS PV)
    # SCAN_ID_PV: "IOC:bluesky_scan_id"

    ### Where to "autosave" the RE.md dictionary.
    ### Defaults:
    MD_PATH: .re_md_dict.yml

    ### The progress bar is nice to see,
    ### except when it clutters the output in Jupyter notebooks.
    ### Default: False
    USE_PROGRESS_BAR: false

### Baseline stream
### When ENABLE=true, all ophyd objects with a "baseline" label
### will be added to the baseline stream.
BASELINE_LABEL:
    ENABLE: true

### Best Effort Callback Configurations
### Defaults: all true
### except no plots in queueserver
BEC:
    BASELINE: true
    HEADING: true
    PLOTS: false
    TABLE: true

### Support for known output file formats.
### Uncomment to use.  If undefined, will not write that type of file.
### Each callback should apply its configuration from here.
NEXUS_DATA_FILES:
    ENABLE: false
    FILE_EXTENSION: hdf

SPEC_DATA_FILES:
    ENABLE: true
    FILE_EXTENSION: dat

### APS Data Management
### Learn environment variables for Data Management from this file:
DM_SETUP_FILE: "/home/dm/etc/dm.setup.sh"

# ----------------------------------

OPHYD:
    ### Control layer for ophyd to communicate with EPICS.
    ### Default: PyEpics
    ### Choices: "PyEpics" or "caproto"
    CONTROL_LAYER: PyEpics

    ### default timeouts (seconds)
    TIMEOUTS:
        PV_READ: &TIMEOUT 5
        PV_WRITE: *TIMEOUT
        PV_CONNECTION: *TIMEOUT

XMODE_DEBUG_LEVEL: Plain

```

## Real-time Data Analysis

### 1. Live Data Inspection

```python
import datetime

# During or after scans, examine data
run = cat[-1]  # Get most recent run

# Basic run information
print(f"Scan ID: {run.metadata['start']['scan_id']}")
print(f"Plan name: {run.metadata['start']['plan_name']}")
# This is a floating-point time (always in UTC):
print(f"Start time: {run.metadata['start']['time']}")
# This is human-readable time (local time zone when defined):
print(f"Start time: {datetime.datetime.fromtimestamp(run.metadata['start']['time'])}")

# Read data
data = run.primary.read()
print(f"Data variables: {list(data)}")

# Quick plot (if matplotlib available)
try:
    import matplotlib.pyplot as plt, datetime
    
    # Get motor and detector data.
    # NOTE: for simulated devices the ophyd ".name" (e.g. "sim_motor") differs
    # from the data key (e.g. "motor"), so read the key from ".hints".
    motor_key = sim_motor.hints["fields"][0]     # -> "motor"
    detector_key = sim_det.hints["fields"][0]    # -> "noisy_det"
    motor_data = data[motor_key]
    detector_data = data[detector_key]
    plan_name = run.metadata["start"].get("plan_name")
    scan_id = run.metadata["start"]["scan_id"]  # If scan_id not available, use 'uid'
    start_time = datetime.datetime.fromtimestamp(run.metadata['start']['time'])
    
    plt.figure()
    plt.plot(motor_data, detector_data, 'o-')
    plt.xlabel(f'{motor_key} position')
    plt.ylabel(f'{detector_key} counts')
    supertitle = f'Scan {scan_id}'
    if plan_name is not None:
        supertitle += f" ({plan_name!r})"
    plt.suptitle(supertitle)
    plt.title(f'started {start_time}')
    plt.show()
    
except ImportError:
    print("Matplotlib not available for plotting")
```

### 2. Data Export

```python
# Export to CSV
import pandas as pd

# Convert to DataFrame
df = data.to_dataframe()
print(df.head())

# Save to file
filename = f"scan_{run.metadata['start']['scan_id']}.csv"
df.to_csv(filename)
print(f"Data saved to {filename}")
```

## Advanced Interactive Techniques

### 1. Custom Metadata

```python
# Add custom metadata to scans
custom_md = {
    'sample': 'test_sample_001',
    'temperature': 295.0,
    'operator': 'your_name',
    'notes': 'Testing new setup'
}

# RE returns list of run uids.  Keep the first one for ...
uid, = RE(bp.scan([sim_det], sim_motor, -1, 1, 11, md=custom_md))

# Verify metadata was saved using uid reported by RE above.
latest_run = cat[uid]
print("Custom metadata:")
for key, value in custom_md.items():
    print(f"  {key}: {latest_run.metadata['start'].get(key, 'Not found')}")
```

### 2. Multi-step Procedures

```python
# Complex measurement procedure
def measurement_procedure():
    """Multi-step measurement."""
    
    print("Starting measurement procedure...")
    
    # Step 1: Move to a starting position
    print("Step 1: Move sim_motor to start")
    yield from bps.mv(sim_motor, 0)
    
    # Step 2: Shape the simulated detector signal
    print("Step 2: Configure sim_det")
    yield from bps.mv(sim_det.Imax, 10000, sim_det.sigma, 1)
    
    # Step 3: Take measurement
    print("Step 3: Main measurement")
    yield from bp.scan([sim_det], sim_motor, -2, 2, 41)
    
    print("Measurement procedure complete!")

# Execute procedure
RE(measurement_procedure())
```

## Troubleshooting Interactive Sessions

### 1. Common Issues

```python
# Device connection issues
if not sim_motor.connected:
    print("Motor sim_motor not connected!")
    # Simulated devices are always connected; for EPICS devices,
    # check IOC status and the EPICS environment

# RunEngine stuck
if RE.state != 'idle':
    print(f"RunEngine state: {RE.state}")
    # May need RE.abort().  Alternative is RE.stop()

# Memory issues with large datasets
import gc
gc.collect()  # Force garbage collection
```

### 2. Debugging Tools

```python
# Enable verbose logging
import logging
logging.getLogger('bluesky').setLevel(logging.DEBUG)

# Check device details
sim_motor.summary()   # Summarize the 'sim_motor' object
sim_det.describe()    # Detailed detector info

# Monitor device values
sim_motor.subscribe(lambda **kwargs: print(f"Motor moved to {kwargs['value']}"))
```

## Best Practices

### 1. Session Workflow

1. **Start Clean**: Always start with `from my_instrument.startup import *`
2. **Check Status**: Use `%wa` to verify device status
3. **Test First**: Test devices with small moves before big scans
4. **Save Work**: Export important data regularly
5. **Document**: Add meaningful metadata to all scans

### 2. Safety Practices

```python
# Check the current position before large moves
print(f"Current position: {sim_motor.position}")
# For real EpicsMotors you can also check limits with `motor.limits`
# (the simulated sim_motor has no limits).

# Use relative moves for safety
RE(bps.mvr(sim_motor, 0.1))  # Safer than absolute moves

# Check detector reading
RE(bp.count([sim_det], num=1))
# Verify reasonable readings before long scans
```

### 3. Efficiency Tips

```python
# Use IPython features
# - Tab completion: sim_motor.<TAB>
# - Command history: Up arrow, Ctrl+R to search
# - Magic commands: %wa, %ct, %mov

# Move and wait using a plan
RE(bps.mv(sim_motor, 1.0))

# Use meaningful variable names
current_run = cat[-1]
motor_pos = sim_motor.position
```

## Deliverables

After completing this step, you should be able to:

- ✅ Start IPython and load your instrument efficiently
- ✅ Use magic commands for device control and inspection
- ✅ Perform interactive scans and device testing
- ✅ Access and analyze data in real-time
- ✅ Handle common troubleshooting scenarios
- ✅ Document and save your work appropriately

## Next Steps

With interactive operation mastered, you're ready for:
- **Jupyter notebooks** for analysis and documentation
- **Queue server** for remote and automated operation  
- **Advanced data visualization** with specialized tools

**Next Step**: Advanced topics (Queue Server, Data Analysis, Production Deployment)

---

## Reference: Essential IPython Commands

| Command | Purpose | Example |
|---------|---------|---------|
| `%wa` | List all devices | `%wa motors` |
| `%mov` | Move motors | `%mov sim_motor 2.5` |
| `%movr` | Relative move | `%movr sim_motor 0.1` |
| `%ct` | Count detectors | `%ct baseline` |
| `object?` | Get help | `bp.scan?` |
| `object.<TAB>` | Tab completion | `sim_motor.<TAB>` |
| `%hist` | Command history | `%hist -n 10` |

## Common Scan Patterns

| Pattern | Command | Use Case |
|---------|---------|----------|
| Point measurement | `bp.count([sim_det], num=5)` | Detector characterization |
| Linear scan | `bp.scan([sim_det], sim_motor, -1, 1, 21)` | Response curves |
| Relative scan | `bp.rel_scan([sim_det], sim_motor, -0.5, 0.5, 11)` | Local optimization |
| List scan | `bp.list_scan([sim_det], sim_motor, positions)` | Specific points |
| Grid scan | `bp.grid_scan([sim_det], sim_motor, -1, 1, 5, sim_motor2, -1, 1, 5)` | 2D mapping (needs a 2nd motor) |