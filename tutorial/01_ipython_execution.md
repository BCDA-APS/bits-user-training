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

# Move motors (relative)
%mov m1 1.5          # Move m1 to (absolute) position 1.5
%movr m1 0.1         # Move m1 (relative) by +0.1
%movr m1 -0.1 m2 0.2 # Move (relative) multiple motors, m1 by -0.1, m2 by +0.2

# Count (labeled devices)
%ct                  # Count devices with "detectors" label (uses their configured count times)
%ct baseline         # Count devices with "baseline" label  
%ct monitors         # Count devices with "monitors" label
```


### Read

command | description
--- | ---
`OBJECT.get()` | low-level command to show value of ophyd *Signal* named `OBJECT`
`OBJECT.read()` | data acquisition command, includes timestamp
`listdevice(OBJECT)` | table-version of `.read()`
`MOTOR.position` | get readback, only for motor objects
`MOTOR.user_readback.get()` | alternative to `MOTOR.position`
`OBJECT.summary()` | more information about `OBJECT`

<details>
<summary>Examples:</summary>

<pre>
In [10]: <b>m1.user_setpoint.get()</b>
Out[10]: 0.0

In [11]: <b>m1.user_setpoint.read()</b>
Out[11]: {'m1_user_setpoint': {'value': 0.0, 'timestamp': 1613878949.1092}}

In [12]: <b>listdevice(m1)</b>
================ ===== ==========================
name             value timestamp
================ ===== ==========================
m1               0.0   2021-02-20 21:42:29.109200
m1_user_setpoint 0.0   2021-02-20 21:42:29.109200
================ ===== ==========================

Out[12]: <pyRestTable.rest_table.Table at 0x7fe0649cbd00>

In [13]: <b>m1.position</b>
Out[13]: 0.0

In [14]: <b>m1.summary()</b>
data keys (* hints)
-------------------
*m1
 m1_user_setpoint

read attrs
----------
user_readback        EpicsSignalRO       ('m1')
user_setpoint        EpicsSignal         ('m1_user_setpoint')

config keys
-----------
m1_acceleration
m1_motor_egu
m1_user_offset
m1_user_offset_dir
m1_velocity

configuration attrs
-------------------
user_offset          EpicsSignal         ('m1_user_offset')
user_offset_dir      EpicsSignal         ('m1_user_offset_dir')
velocity             EpicsSignal         ('m1_velocity')
acceleration         EpicsSignal         ('m1_acceleration')
motor_egu            EpicsSignal         ('m1_motor_egu')

unused attrs
------------
offset_freeze_switch EpicsSignal         ('m1_offset_freeze_switch')
set_use_switch       EpicsSignal         ('m1_set_use_switch')
motor_is_moving      EpicsSignalRO       ('m1_motor_is_moving')
motor_done_move      EpicsSignalRO       ('m1_motor_done_move')
high_limit_switch    EpicsSignalRO       ('m1_high_limit_switch')
low_limit_switch     EpicsSignalRO       ('m1_low_limit_switch')
high_limit_travel    EpicsSignal         ('m1_high_limit_travel')
low_limit_travel     EpicsSignal         ('m1_low_limit_travel')
direction_of_travel  EpicsSignal         ('m1_direction_of_travel')
motor_stop           EpicsSignal         ('m1_motor_stop')
home_forward         EpicsSignal         ('m1_home_forward')
home_reverse         EpicsSignal         ('m1_home_reverse')
soft_limit_lo        EpicsSignal         ('m1_soft_limit_lo')
soft_limit_hi        EpicsSignal         ('m1_soft_limit_hi')
steps_per_rev        EpicsSignal         ('m1_steps_per_rev')
</pre>

</details>

## Move

command | description
--- | ---
`%mov MOTOR value` | interactive command move MOTOR to value (command line only)
`%movr MOTOR value` | interactive command relative move (command line only)
`MOTOR.move(value)` | ophyd command to `%mov`
`MOTOR.user_setpoint.put(value)` | ophyd to set motor `.VAL` field but not wait
`bps.mv(MOTOR, value)` | bluesky plan command to move and wait for completion
`bps.mv(MOTOR.user_setpoint, value)` | bluesky plan command, same as above
`bps.mvr(MOTOR, value)` | bluesky plan command, relative move

<details>
<summary>Examples:</summary>

<pre>
In [15]: <b>%mov m1 1</b>

In [16]: <b>%movr m1 -1</b>

In [17]: <b>m1.move(.5)</b>
Out[17]: MoveStatus(done=True, pos=m1, elapsed=0.8, success=True, settle_time=0.0)

In [18]: <b>m1.user_setpoint.put(1)</b>

In [19]: <b>RE(bps.mv(m1, 0))</b>
Out[19]: ()
</pre>

</details>


## Count

command | description
--- | ---
`%ct` | count _all_ objects with label `detectors` and format output (command line only)
`SCALER.trigger().wait(); SCALER.read()` | ophyd command to count SCALER
`bp.count([SCALER])` | bluesky plan to count

Count time setting is different for various types of detectors:

detector | set count time
--- | ---
scaler | `SCALER.preset_time.put(COUNT_TIME_S)`
area detector | `AD.cam.acquire_time.put(COUNT_TIME_S)`

<details>
<summary>Examples:</summary>

<pre>
In [20]: <b>scaler1.preset_time.get()</b>
Out[20]: 1.0

In [21]:<b>%mov scaler1.preset_time 2.5</b>

In [22]: <b>scaler1.preset_time.get()</b>
Out[22]: 2.5

In [23]: <b>%ct</b>
[This data will not be saved. Use the RunEngine to collect data.]
noisy                          68.56615083963807
I0Mon                          12.0
ROI1                           0.0
ROI2                           0.0
scaler_time                    2.6

In [24]: <b>scaler1.trigger().wait()</b>

In [25]: <b>scaler1.read()</b>
Out[25]:
OrderedDict([('I0Mon', {'value': 12.0, 'timestamp': 1613880362.609086}),
             ('ROI1', {'value': 0.0, 'timestamp': 1613880362.609086}),
             ('ROI2', {'value': 0.0, 'timestamp': 1613880362.609086}),
             ('scaler_time', {'value': 2.6, 'timestamp': 1613880338.961804})])

In [26]: <b>scaler1.trigger().wait(); scaler1.read()</b>
Out[26]:
OrderedDict([('I0Mon', {'value': 11.0, 'timestamp': 1613880389.315847}),
             ('ROI1', {'value': 0.0, 'timestamp': 1613880389.315847}),
             ('ROI2', {'value': 0.0, 'timestamp': 1613880389.315847}),
             ('scaler_time', {'value': 2.6, 'timestamp': 1613880362.609086})])
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
motor
  Positioner                     Value       Low Limit   High Limit  Offset
  m1                             0.0         -32000.0    32000.0     0.0
  m2                             0.0         -32000.0    32000.0     0.0
  m3                             0.0         -32000.0    32000.0     0.0
  m4                             0.0         -32000.0    32000.0     0.0
  m5                             0.0         -32000.0    32000.0     0.0
  m6                             0.0         -32000.0    32000.0     0.0
  m7                             0.0         -32000.0    32000.0     0.0
  m8                             0.0         -32000.0    32000.0     0.0

  Local variable name                    Ophyd name (to be recorded as metadata)
  m1                                     m1
  m2                                     m2
  m3                                     m3
  m4                                     m4
  m5                                     m5
  m6                                     m6
  m7                                     m7
  m8                                     m8

detectors
  Local variable name                    Ophyd name (to be recorded as metadata)
  noisy                                  noisy
  scaler                                 scaler

counter
  Local variable name                    Ophyd name (to be recorded as metadata)
  I0
  I0Mon                                  I0Mon
  ROI1                                   ROI1
  ROI2                                   ROI2
  clock
  diode
  scaler.channels.chan08.s               I0Mon
  scaler.channels.chan10.s               ROI1
  scaler.channels.chan11.s               ROI2
  scint


In [44]: <b>listobjects()</b>
====== =============== =============== =========
name   ophyd structure EPICS PV        label(s)
====== =============== =============== =========
I0     EpicsSignalRO   sky:scaler1.S2  counter
I0Mon  EpicsSignalRO   sky:scaler1.S8  counter
ROI1   EpicsSignalRO   sky:scaler1.S10 counter
ROI2   EpicsSignalRO   sky:scaler1.S11 counter
_2     EpicsSignal     sky:scaler1.CNT
clock  EpicsSignalRO   sky:scaler1.S1  counter
diode  EpicsSignalRO   sky:scaler1.S5  counter
m1     MyMotor         sky:m1          motor
m2     MyMotor         sky:m2          motor
m3     MyMotor         sky:m3          motor
m4     MyMotor         sky:m4          motor
m5     MyMotor         sky:m5          motor
m6     MyMotor         sky:m6          motor
m7     MyMotor         sky:m7          motor
m8     MyMotor         sky:m8          motor
mover2 EpicsSignal     IOC:float2
noisy  EpicsSignalRO   sky:userCalc1   detectors
scaler ScalerCH        sky:scaler1     detectors
scint  EpicsSignalRO   sky:scaler1.S3  counter
====== =============== =============== =========

Out[44]: <pyRestTable.rest_table.Table at 0x7fe064171fd0>

In [45]: <b>listruns()</b>
catalog name: bs2021
========= ========================== ======= ======= ========================================
short_uid date/time                  exit    scan_id command
========= ========================== ======= ======= ========================================
e070882   2021-02-06 22:50:08.118423 success 131     rel_scan(detectors=['noisy'], num=19 ...
15621a3   2021-02-06 22:49:58.051389 success 130     rel_scan(detectors=['noisy'], num=19 ...
7322f2f   2021-02-06 22:47:39.789684 success 129     rel_scan(detectors=['noisy'], num=19 ...
02732c2   2021-02-06 22:47:28.456452 success 128     rel_scan(detectors=['noisy'], num=19 ...
1a7f0ce   2020-12-29 22:54:57.604267 success 127     scan(detectors=['fourc'], num=41, ar ...
7dd58eb   2020-12-29 22:46:09.629373 success 126     scan(detectors=['fourc'], num=41, ar ...
d1f5f4f   2020-12-29 22:36:20.358277 success 125     scan(detectors=['fourc'], num=41, ar ...
0f6eac8   2020-12-29 22:34:22.757687 success 124     scan(detectors=['fourc'], num=41, ar ...
23a642d   2020-12-16 22:08:17.257659 success 123     scan(detectors=['fourc_h', 'fourc_k' ...
e89dbed   2020-12-16 22:08:03.778558 success 122     scan(detectors=['fourc_h', 'fourc_k' ...
699c827   2020-12-16 22:07:08.838917 success 121     scan(detectors=['fourc_h', 'fourc_k' ...
978ec2b   2020-12-16 21:00:33.380914 success 120     rel_scan(detectors=['noisy'], num=19 ...
bb22936   2020-12-16 20:59:58.870435 success 119     scan(detectors=['noisy'], num=19, ar ...
3c04995   2020-12-16 20:58:43.471627 success 118     count(detectors=['scaler'], num=1)
========= ========================== ======= ======= ========================================

Out[45]: <pyRestTable.rest_table.Table at 0x7fe064174190>

In [48]: <b>scaler.describe()</b>
Out[48]:
OrderedDict([('I0Mon',
              {'source': 'PV:sky:scaler1.S8',
               'dtype': 'number',
               'shape': [],
               'units': '',
               'lower_ctrl_limit': 0.0,
               'upper_ctrl_limit': 0.0,
               'precision': 0}),
             ('ROI1',
              {'source': 'PV:sky:scaler1.S10',
               'dtype': 'number',
               'shape': [],
               'units': '',
               'lower_ctrl_limit': 0.0,
               'upper_ctrl_limit': 0.0,
               'precision': 0}),
             ('ROI2',
              {'source': 'PV:sky:scaler1.S11',
               'dtype': 'number',
               'shape': [],
               'units': '',
               'lower_ctrl_limit': 0.0,
               'upper_ctrl_limit': 0.0,
               'precision': 0}),
             ('scaler_time',
              {'source': 'PV:sky:scaler1.T',
               'dtype': 'number',
               'shape': [],
               'units': '',
               'lower_ctrl_limit': 0.0,
               'upper_ctrl_limit': 0.0,
               'precision': 3})])

In [49]: <b>scaler.summary()</b>
data keys (* hints)
-------------------
*I0Mon
*ROI1
*ROI2
 scaler_time

read attrs
----------
channels             Channels            ('scaler_channels')
channels.chan08      ScalerChannel       ('scaler_channels_chan08')
channels.chan08.s    EpicsSignalRO       ('I0Mon')
channels.chan10      ScalerChannel       ('scaler_channels_chan10')
channels.chan10.s    EpicsSignalRO       ('ROI1')
channels.chan11      ScalerChannel       ('scaler_channels_chan11')
channels.chan11.s    EpicsSignalRO       ('ROI2')
time                 EpicsSignal         ('scaler_time')

config keys
-----------
scaler_auto_count_delay
scaler_auto_count_time
scaler_channels_chan08_chname
scaler_channels_chan08_gate
scaler_channels_chan08_preset
scaler_channels_chan10_chname
scaler_channels_chan10_gate
scaler_channels_chan10_preset
scaler_channels_chan11_chname
scaler_channels_chan11_gate
scaler_channels_chan11_preset
scaler_count_mode
scaler_delay
scaler_egu
scaler_freq
scaler_preset_time

configuration attrs
-------------------
channels             Channels            ('scaler_channels')
channels.chan08      ScalerChannel       ('scaler_channels_chan08')
channels.chan08.chname EpicsSignal         ('scaler_channels_chan08_chname')
channels.chan08.preset EpicsSignal         ('scaler_channels_chan08_preset')
channels.chan08.gate EpicsSignal         ('scaler_channels_chan08_gate')
channels.chan10      ScalerChannel       ('scaler_channels_chan10')
channels.chan10.chname EpicsSignal         ('scaler_channels_chan10_chname')
channels.chan10.preset EpicsSignal         ('scaler_channels_chan10_preset')
channels.chan10.gate EpicsSignal         ('scaler_channels_chan10_gate')
channels.chan11      ScalerChannel       ('scaler_channels_chan11')
channels.chan11.chname EpicsSignal         ('scaler_channels_chan11_chname')
channels.chan11.preset EpicsSignal         ('scaler_channels_chan11_preset')
channels.chan11.gate EpicsSignal         ('scaler_channels_chan11_gate')
count_mode           EpicsSignal         ('scaler_count_mode')
delay                EpicsSignal         ('scaler_delay')
auto_count_delay     EpicsSignal         ('scaler_auto_count_delay')
freq                 EpicsSignal         ('scaler_freq')
preset_time          EpicsSignal         ('scaler_preset_time')
auto_count_time      EpicsSignal         ('scaler_auto_count_time')
egu                  EpicsSignal         ('scaler_egu')

unused attrs
------------
count                EpicsSignal         ('scaler_count')
update_rate          EpicsSignal         ('scaler_update_rate')
auto_count_update_rate EpicsSignal         ('scaler_auto_count_update_rate')

</pre>

</details>

## Bluesky Plans _vs_. Command-line Actions

There is a difference in the commands to use depending on the context.

context | blocking? | command style
--- | --- | ---
plan function | NOT allowed | call Bluesky [plans](https://blueskyproject.io/bluesky/plans.html) written as [generator](https://wiki.python.org/moin/Generators) functions using `yield from a_plan()`
command line | allowed | use magics (such as `%mov`), `.put()`, and/or `RE(a_plan())`

<details>
<summary>Examples:</summary>

<b>plan function</b>

Write a plan to insert the filters:

```py
def insertFilters(a, b):
    """
    plan: insert the EPICS-specified filters.

    Also, ensure that the two filter positions will be integers.
    """
    yield from bps.mv(pf4_AlTi.fPosA, int(a), pf4_AlTi.fPosB, int(b))
    yield from bps.sleep(0.5)       # allow all filters to re-position
```

Then, call `insertFilters()` from another plan such as

```py
    yield from insertFilters(0, 0)
```

<b>command line actions</b>

There are (at least) three different ways to insert the filters from the command
line:

```py
# use bluesky Magick command
%mov pf4_AlTi.fPosA int(a) pf4_AlTi.fPosB int(b)

# or use ophyd object
pf4_AlTi.fPosA.put(int(a))
pf4_AlTi.fPosB.put(int(b))

# or use the bluesky RunEngine
RE(bps.mv(pf4_AlTi.fPosA, int(a), pf4_AlTi.fPosB, int(b)))
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
print(f"Motor m1 position: {m1.position}")
print(f"Motor limits: {m1.limits}")
print(f"Motor moving: {m1.moving}")

# Test small moves
initial_pos = m1.position
print(f"Starting at: {initial_pos=}")

# Move relative
RE(bps.mvr(m1, 0.1))
print(f"After +0.1: {m1.position=}")

# Move back
RE(bps.mvr(m1, -0.1))
print(f"Back to: {m1.position=}")

# Absolute move
RE(bps.mv(m1, 0.0))
print(f"At zero: {m1.position=}")
```

### 2. Detector Testing

```python
# Test detector reading
print(f"Scaler connected: {scaler1.connected}")
print(f"Current counts: {scaler1.channels.read()}")

# Simple count
RE(bp.count([scaler1], num=1, delay=1))

# Count with multiple detectors
RE(bp.count([scaler1, scaler2], num=3, delay=2))

# Area detector test (if available)
if 'simdet' in globals():
    print(f"Area detector connected: {simdet.connected}")
    RE(bp.count([simdet], num=1))

# Setting count times for different detector types
# Scaler count time
RE(bps.mv(scaler1.preset_time, 0.1))  # 0.1 s counting time

# Area detector count time
if 'simdet' in globals():
    RE(bps.mv(simdet.cam.acquire_time, 0.1))  # 0.1 s counting time
```

### 3. Combined Device Testing

```python
# Test motor and detector together
print("Testing coordinated motion and detection...")

# Move motor and count at each position
positions = [-0.5, 0, 0.5]
for pos in positions:
    print(f"\nMoving to {pos}")
    RE(bps.mv(m1, pos))
    
    print(f"Counting at {m1.position=}")
    RE(bp.count([scaler1], num=1))
```

## Interactive Scanning

### 1. Basic Scans

```python
# Simple scan
RE(bp.scan([scaler1], m1, -1, 1, 11))

# Relative scan (around current position)
RE(bp.rel_scan([scaler1], m1, -0.5, 0.5, 11))

# List scan (specific positions)
positions = [-1, -0.25, 0, 0.35, 1]  # note irregular spacing
RE(bp.list_scan([scaler1], m1, positions))

# Count without motion
RE(bp.count([scaler1], num=5, delay=1))
```

### 2. Multi-dimensional Scans

```python
# Grid scan with 2 motors
RE(bp.grid_scan([scaler1], 
                m1, -1, 1, 5,    # m1: 5 points from -1 to 1
                m2, -0.5, 0.5, 3, # m2: 3 points from -0.5 to 0.5
                snake_axes=[m2]))  # Snake pattern in m2

# Grid scan with 2 motors and relative positions.
RE(bp.rel_grid_scan([scaler1],
                        m1, -1, 1, 5,
                        m2, -0.5, 0.5, 3))
```

### 3. Using Your Custom Plans

```python
# Motor characterization
RE(motor_characterization(m1, scaler1, num_points=21))

# Quick scan around current position
RE(quick_scan(m1, scaler1, range_pm=1.0, points=11))

# Sample alignment
RE(sample_alignment([sample_x, sample_y], scaler1, step_size=0.1))

# Detector optimization
RE(detector_optimization(m1, scaler1, initial_range=2.0))
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
    
    # Get motor and detector data
    motor_data = data[m1.name]
    detector_data = data[f'{scaler1.channels.chan02.s.name}']  # Adjust as needed
    plan_name = run.metadata["start"].get("plan_name")
    scan_id = run.metadata["start"]["scan_id"]  # If scan_id not available, use 'uid'
    start_time = datetime.datetime.fromtimestamp(run.metadata['start']['time'])
    
    plt.figure()
    plt.plot(motor_data, detector_data, 'o-')
    plt.xlabel(f'{m1.name} position')
    plt.ylabel(f'{scaler1.name} counts')
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
uid, = RE(bp.scan([scaler1], m1, -1, 1, 11, md=custom_md))

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
    """Multi-step measurement with error handling."""
    
    print("Starting measurement procedure...")
    
    # Step 1: Align sample
    print("Step 1: Sample alignment")
    yield from sample_alignment([sample_x, sample_y], scaler1)
    
    # Step 2: Optimize detector
    print("Step 2: Detector optimization")
    yield from detector_optimization(m1, scaler1)
    
    # Step 3: Take measurement
    print("Step 3: Main measurement")
    yield from bp.scan([scaler1], m1, -2, 2, 41)
    
    print("Measurement procedure complete!")

# Execute procedure
RE(measurement_procedure())
```

## Troubleshooting Interactive Sessions

### 1. Common Issues

```python
# Device connection issues
if not m1.connected:
    print("Motor m1 not connected!")
    # Check IOC status, EPICS environment

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
m1.summary()     # Summarize the 'm1' object
scaler1.describe() # Detailed detector info

# Monitor device values
m1.subscribe(lambda **kwargs: print(f"Motor moved to {kwargs['value']}"))
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
# Always check motor limits before large moves
print(f"Motor limits: {m1.limits}")
print(f"Current position: {m1.position}")

# Use relative moves for safety
RE(bps.mvr(m1, 0.1))  # Safer than absolute moves

# Check detector count rates
RE(bp.count([scaler1], num=1))
# Verify reasonable count rates before long scans
```

### 3. Efficiency Tips

```python
# Use IPython features
# - Tab completion: motor1.<TAB>
# - Command history: Up arrow, Ctrl+R to search
# - Magic commands: %wa, %ct, %mov

# Combine operations efficiently
RE(bps.mv(m1, 1.0, m2, 2.0))  # Move multiple motors simultaneously

# Use meaningful variable names
current_run = cat[-1]
motor_pos = m1.position
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
| `%mov` | Move motors | `%mov m1 2.5` |
| `%movr` | Relative move | `%movr m1 0.1` |
| `%ct` | Count detectors | `%ct baseline` |
| `object?` | Get help | `bp.scan?` |
| `object.<TAB>` | Tab completion | `motor1.<TAB>` |
| `%hist` | Command history | `%hist -n 10` |

## Common Scan Patterns

| Pattern | Command | Use Case |
|---------|---------|----------|
| Point measurement | `bp.count([det], num=5)` | Detector characterization |
| Linear scan | `bp.scan([det], motor, -1, 1, 21)` | Response curves |
| Relative scan | `bp.rel_scan([det], motor, -0.5, 0.5, 11)` | Local optimization |
| List scan | `bp.list_scan([det], motor, positions)` | Specific points |
| Grid scan | `bp.grid_scan([det], m1, -1, 1, 5, m2, -1, 1, 5)` | 2D mapping |