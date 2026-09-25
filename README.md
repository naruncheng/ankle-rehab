# IMU Logger

Python serial logger for the ESP32-S3 firmware in `imu_stream.ino`, which
streams 3x MPU6050 IMU data as CSV lines at ~100 Hz over USB serial.

## Setup

```bash
pip install -r requirements.txt
```

## Usage

Auto-detect/prompt for the serial port:

```bash
python imu_logger.py --subject S01 --condition walk
```

Specify the port directly:

```bash
python imu_logger.py --port COM9 --subject S01 --condition walk --trial 2
```

Custom IMU-to-body-segment mapping:

```bash
python imu_logger.py --subject S01 --condition walk --segments "imu1=pelvis,imu2=trunk,imu3=shank"
```

### What happens on startup

1. Connects to the board, prints any `#` firmware status lines (e.g. which
   IMUs were found).
2. Calibrates gyro bias: keep all IMUs still for `--calib-seconds` (default
   2s). The per-IMU mean gyro reading is recorded in the trial's
   `_meta.json` — the raw data file is never modified.
3. Opens `./data/{date}_{subject}_{trial:02d}_{condition}.csv` (auto-
   incrementing the trial number if that file already exists) and starts
   recording immediately.

### While running

- `0`-`9` — set the current label (written into every row from then on).
  Edit the `LABEL_MAP` dict at the top of `imu_logger.py` to name your
  labels (default: `0=unlabeled, 1=balanced, 2=perturbation, 3=recovery`).
- `SPACE` — pause/resume recording (serial data keeps being read either
  way, it's just not written to disk while paused).
- `q` (or Ctrl+C) — stop and save cleanly.

The console shows a live status line with sample rate, current label,
recording state, and dropped/malformed line count. A warning is printed if
any IMU's data has been `NaN` for more than 1 second (e.g. disconnected or
misbehaving).

## Output files

For each trial, three files are written to `./data/`:

- `{stem}.csv` — raw samples: `pc_time, t_ms, label` + 18 IMU columns
  (`imu1_ax, imu1_ay, imu1_az, imu1_gyro_x, imu1_gyro_y, imu1_gyro_z, ...`).
  Missing/failed readings are empty cells (read back as `NaN` by pandas).
- `{stem}_meta.json` — subject, condition, trial, start/end time, port,
  baud, sample rate, accel/gyro ranges, segment map, gyro bias offsets,
  firmware status lines, and final sample/drop counts.
- `{stem}_events.csv` — every label change, with `pc_time, t_ms, label`.

## Loading a trial in pandas

```bash
python load_trial.py data/20260925_S01_01_walk.csv
```

Or from code:

```python
from load_trial import load_trial

df, meta = load_trial("data/20260925_S01_01_walk.csv")
```

`load_trial()` returns the CSV as a DataFrame with the recorded gyro bias
offsets subtracted from the `imu{n}_gyro_{x,y,z}` columns, plus the parsed
metadata dict.
