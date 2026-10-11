# rasta-dataprocess

Post-flight tools for viewing, calibrating and exporting UCATS-B logs. The
instrument is now named RASTA (Rapid Analyzer for Stratospheric Transport and
Age of air).

This repo only reads CSV files that already exist. The `ucats-b` repo
(`~/code/ucats-b`) runs on the aircraft and produces them. Here you can plot
them, clean them, calibrate them against the onboard cal tanks, and write
files for delivery.

Which detector measures which gas changes between flights. The viewer reads
each file's header and offers only the gases that file contains.

## Install

Python 3.9+ and the packages in `requirements.txt`:

```
pip install -r requirements.txt
```

## Run

```
python3 rasta_gui.py [/path/to/ucatsb-YYYYMMDDHH.csv]
```

The file is optional. You can start empty and use **Load Data** instead.

## The window

| Tab | What it is for |
|---|---|
| **Timeseries** | One gas against time, with the calibrated record overlaid. Has the Stats and Flag tools. |
| **Calibration** | Diagnostics: how good is the calibration for this gas? |
| **Correlations** | One calibrated tracer against another. |
| **Cal Tanks** | Which two cal tanks flew on this flight. |
| **Export** | Writes the CSV and ICARTT files. |

The left panel is shared by Timeseries, Calibration, Cal Tanks and Export.
Correlations has its own panel, because it is about two gases at once.

Zoom and pan survive any settings change. Only **Home**, a gas change and a
cal-tank change reset the view.

## Left panel

Settings are kept per gas, except the cal tanks, which are per flight.

| Control | What it does |
|---|---|
| **Load Data** | Open a file, or pick from recently opened ones. A file that has moved is greyed out as `(missing)`. The list is stored in `.rasta_gui_state.yaml`. |
| **Save…** | Writes the settings to a config file. A `•` means unsaved changes. See [Per-flight settings](#per-flight-settings). |
| **Gas** | CO2, N2O, CH4, O3 or H2O, whichever the file has. On the Timeseries tab this is linked to the Correlations X axis. |
| **Above / Other / Right axis** | Add a small panel above the main plot (detector pressure, cell temperature, or any column in the file) and optionally a second trace on a right-hand axis. |
| **Info notes** | Show or hide the grey key of notes on the figure. Off by default. |
| **Trim (min)** | Exclude the first *N* minutes (warm-up) and the last *M* minutes (descent) of the record. |
| **Pressure ±mbar** | Exclude rows where detector pressure is more than this far from 140 mbar. |
| **Pumps on** | Keep only rows with the sample pumps running. Leave it off for bench runs, which are pumps-off throughout. |
| **Smooth P** | Averages the detector pressure over this many seconds before **Pdelt** divides by it, so pressure noise is not added to the mole fraction. 10–30 s is enough. |
| **Correct: Pdelt, Tdelt** | Scale each measurement to the detector's spec conditions (140 mbar, 40 °C) *before* calibrating. Both use that gas's own detector. |
| **Flag Air** | Drop *N* seconds of air after each cal injection, while the cell clears. See [Rows that are not air](#rows-that-are-not-air). |
| **Copy settings to all gases** | Copies the trim, pressure, Smooth P, Pumps on, Flag Air and cal-window values to the other gases. Pdelt/Tdelt, drift model and smoothing stay as they are. |
| **Calibration: Window** | Per tank, which part of each injection is averaged. Offsets in seconds from the injection's last row, e.g. `-15` to `-1`. |
| **Calibration: Model** | How the calibration follows the tanks in time. See [Calibration](#calibration). |
| **Smoothing: Box smooth** | None or 2–6 point average of the delivered record. See [Smoothing](#smoothing). |
| **Flagged Points** | Count of hand-flagged points, with Undo and Clear. See [Flagging](#flagging-bad-points-flag). |

Trim, pressure and Pumps on are applied **before** the cal means are
computed, so they change the calibration, not just the picture. A cal point
disappears if its window has no data left.

Pdelt and Tdelt also change the calibration, on purpose: a cal injection
measured at 138 mbar is put on the same footing as air measured at 140 before
it becomes a calibration point. They are off by default.

## Per-flight settings

Settings belong to the flight, because the right warm-up, cal windows and
cal tanks are properties of that flight. They live in a YAML file beside the
CSV, named `<dataset>_conf.yaml`:

```yaml
cals:
  cal0: CC302489
  cal1: CB09960
flagged:                 # points struck out by hand, as raw-file row ranges
  Ozone:
    rows: 16199          # the CSV length they were drawn against
    ranges:
      - [4271, 4272]
smoothing:               # box smooth in points, per gas
  CO2: 3
  Ozone: 5
CO2:
  warmup_min: 12
  pressure_tol_mbar: 0.3
  flag_air_s: 38
  cal1_window_s: [-8, -1]
  cal2_window_s: [-8, -1]
  drift_model: linear
N2O:
  ...
```

- **Nothing is written until you press Save.** Opening a file, experimenting
  and quitting leaves everything as it was.
- Save always asks for a name. Use a different one to keep several
  configurations of the same flight (`..._tight_conf.yaml`).
- Quitting, or loading another file, with unsaved changes asks Save / Don't
  Save / Cancel.
- Opening a file with one config applies it. With several, a chooser appears.
  A file with none starts from defaults. **File > Load Configuration…** opens
  one by name.
- There is no template carried from one flight to the next. A new flight
  always starts from defaults.

## Timeseries tools

Both are toggles on the right of the Timeseries toolbar.

### Box statistics (Stats)

Drag a box over some data to get its sample count, mean, standard deviation
and range, with a **Copy** button.

- No masking is applied. The box is how you say which data you mean.
- A combo box picks which plotted trace the numbers are for. If that trace is
  on a different axis from the box, only the time span is used and the
  readout says so.
- The count of points inside the time span but outside the box vertically is
  always shown. A box that clips the data narrows the reported spread.

### Flagging bad points (Flag)

For readings that are plainly wrong but that no rule catches, such as ozone
spikes to 3500 ppb on a 0–900 ppb record.

- **Left-drag** a box to flag, **right-drag** to unflag.
- Flagged points are marked with a black `x` and removed from the calibrated
  (or, for O3 and H2O, filtered) record, from Correlations, and from both
  exports. The raw blue trace still shows everything.
- The box is matched against the **raw** trace, so a flag means the same rows
  after you change the drift model. A tight box around the red calibrated
  line can therefore catch nothing, and the readout says so.
- Unflagging ignores the box height and clears the whole time span, because a
  flagged spike is usually off the top of the axis.
- On a calibrated gas, a flagged row inside a cal window is dropped before
  that injection's mean is computed. Flagging a bad injection changes the
  calibration.
- **Apply to all gases** spreads a new flag across every gas, for an inlet or
  pump problem. **Clear** only affects the current gas. **Undo** lasts for
  this session.
- The same tool works on the Correlations tab, where a bad point is often
  easier to spot. See [Correlations](#correlations).

Flags are saved with the flight, but only when you press Save. They are
stored as row ranges, so thousands of points cost a few lines.

## Cal Tanks tab

Choose which two tanks were plumbed in for **this flight**. `cals.yaml` lists
the tanks in use *now*, so it is probably wrong for an older flight, and the
wrong assigned values bias every calibrated number.

- Both combos list every tank in `cals.yaml`. The assigned values for each
  gas are shown beside them so you can check the pick.
- Warnings appear for a pair that cannot work: a tank missing from the roster,
  the same tank twice, or a tank with no value for a gas in the file.
- The order of the two combos does not matter. Which tank is flowing is
  worked out from the measured concentration. Only the *pair* has to be right.
- **Reset to cals.yaml default** restores the `cals:` pairing.

## Calibration

The Calibration tab turns the cal-tank injections into a time-varying
two-point calibration for one gas, and shows how far to trust it.

### How it is built

Two tanks of known mole fraction are injected repeatedly during the flight.
Each injection is averaged (using the cal window) into one measured response,
`R`, for that tank at that time. These are the dots on the Timeseries.
Interpolating each tank's response through time gives, at every ambient row:

```
slope(t)     = (A_hi - A_lo) / (R_hi(t) - R_lo(t))
intercept(t) = A_lo - slope(t) * R_lo(t)
calibrated   = slope(t) * measured(t) + intercept(t)
```

`A` is a tank's assigned value from `cals.yaml`.

- **Drift removal and calibration are one step.** The tanks' own responses
  carry the drift, so there is no separate detrending pass.
- **Both slope and intercept are needed.** The span error is several percent.

The **Model** controls how the responses are interpolated:

| Model | What it does |
|---|---|
| `linear` (default) | Straight lines between injections. Passes each injection's noise into the data. |
| `smooth` | Averages over *N* neighbouring injections first (the `N ev` box). For scatter much larger than real drift. |
| `constant` | One flight-mean response per tank. A baseline: what a single static calibration gives. |
| `fixed slope` | Holds the gain at one number and fits only the intercept. `auto` uses the constant model's slope. |

### Rows that are not air

- **The switch-over sample.** Serial data arrives about 1 s after the gas it
  describes, so the first row flagged as "air" after an injection is still
  tank gas. That row is always blanked from the calibrated record. It does not
  move any cal mean.
- **Flag Air.** The cells keep reading toward the tank for seconds after the
  injection. Flag Air blanks that many seconds from the calibrated record. It
  is 0 by default and does not affect the cal means. The raw trace keeps those
  rows, which is how you judge the length. Flights differ (from ~15 s to
  ~45 s to settle), so set it per flight.

### Reading the panels

1. **Cal bottle response.** Each tank's `measured − assigned`, so zero means
   "reads what it should". A hollow ✕ is a rejected point whose window caught
   the other tank. A whole series far from zero means `cals.yaml` may not
   describe this flight.
2. **Calibration coefficients.** `slope(t)` on the left axis, `intercept(t)`
   on the right. Hatching marks **extrapolated** spans where the calibration
   is held flat. Expect plenty near the ends of a record.
3. **Residuals.**
   - *Filled dot*: closure residual. Zero by construction under `linear`; if
     it is not, something is wrong.
   - *Hollow circle*: leave-one-out. The injection is dropped and predicted
     from the others, which shows how wrong the calibration would be between
     cals. The per-tank RMS in the legend is the main quality number. Look
     for runs and steps, not random scatter.
   - *Shaded band*: the tank's certified uncertainty. It is thin next to the
     scatter, which is the point.
4. **Uncertainty against mole fraction.** The 1σ is smallest between the two
   tanks and grows as the air moves outside them. The shaded band is the
   flight's 1st–99th percentile, and the right axis gives the same curve as a
   percentage of the higher tank.

The header above the panels gives the gas, mode, span gain, the extrapolated
fraction and per-tank counts. Warnings turn it red. The most common is a
**mismatch advisory**: a tank's response is far from its assigned value, and
it names the nearest roster tank. It only prompts you to check the Cal Tanks
tab and never changes the tank itself.

### Using the result

The calibrated series is the **good ambient record only**. Cal periods, the
switch-over sample, Flag Air rows and masked spans are blank, so the line
breaks over them. Nothing is lost, since `cal_slope` and `cal_intercept` are
exported on every row.

The **Calibrated** combo on the Timeseries toolbar overlays this record on
the raw trace:

- **Off**: no overlay. This is the default at startup.
- **Solid**: the record in red.
- **Time / Ozone / Pressure**: colours the points by that variable, with the
  same colours as the Correlations tab.

For O3 and H2O, which are calibrated by their own instruments, the combo shows
the *filtered* record instead (see [Validity floors](#validity-floors)). The
combo is a view setting and is not saved.

### When there is no calibration

The tab shows one sentence saying why, and Export lists the gas as `NOT
calibrated`. This happens for O3 and H2O, when no cal events survive the
masking, and when `cals.yaml` has no value for the gas. A flight with one
usable tank falls back to an offset-only correction and says so.

## Smoothing

**Box smooth** (None, 2pt … 6pt) sits between *Calibration* and *Flagged
Points* on the left panel, and in *Calibration Settings* on the Correlations
tab. It is one setting seen from two places. It is chosen per gas, for all
five, and saved in the flight's `smoothing:` block.

It smooths the record you deliver: the calibrated trace (the filtered one for
O3 and H2O), the Correlations axes and both exports. It is applied last, so
it cannot change the calibration.

- The raw trace, the cal means, slope, intercept and 1σ are not smoothed.
- The exported `<gas>_cal` column is the smoothed one. The slope and
  intercept columns describe the unsmoothed value, and the CSV notes say so.
- The ICARTT header does not mention smoothing. Say so in *Data info* if the
  file should.
- On the Timeseries the effect only shows when **Calibrated** is not Off.

**What happens next to a blank.** A masked span is a wall:

- Blank rows stay blank. Smoothing never fills a gap.
- No average reaches across a blank. Each unbroken stretch of good air is
  smoothed on its own. A jump in the timestamps wider than the window is a
  wall too.
- The window shrinks evenly at the edge of a stretch. The row beside a gap is
  averaged with one neighbour each side, and the edge row keeps its own value.
  The average stays centred, so it does not lag a trend. The cost is that edge
  rows are a little noisier.
- An even count (2, 4, 6) has no middle row, so it is centred half a sample
  early (0.5 s at 1 Hz).

## Correlations

Plots one calibrated tracer against another, e.g. N2O against CO2. Each gas is
analysed with its own saved settings.

**Panel and toolbar**

- **X axis / Y axis / Swap axes**: any two gases in the file. X is linked to
  the Timeseries gas, so changing either changes the other. Y is independent.
- **Marker size, Error bars (1σ), Display cals, Linear fit (OLS), Info
  notes**: display options.
- **Color by** (toolbar): Off, Time, Ozone or Pressure. A colour bar is always
  drawn. The Timeseries tab uses the same encodings.
- **Flagged Points**: the same tool as on the Timeseries. **Flag applies to**
  picks the Y tracer, the X tracer or both, and defaults to Y because an
  ozone spike says nothing about the N2O at that instant. Flagged points stay
  as black `x` so you can unflag them. **Hide flagged points** removes the
  markers without replotting, so your zoom is kept.
- **Calibration Settings**: the same per-gas settings as the other tabs, for
  the tracer chosen in *Tracer*. This is where you set Box smooth for either
  axis.

**What is plotted**

- Each axis is calibrated wherever a calibration exists. A slope taken from
  uncalibrated counts would carry the detector's gain error.
- Ozone has no cal bottles, so it appears as the monitor's own `oz_o3best`
  with below-floor readings removed, and with no error bars. It reports about
  every 2 s, so `n` is lower.
- A point is drawn only where **both** tracers have a value. Each gas's
  masking reaches the other through that intersection.
- The figure notes how many rows remain and how much sits where a calibration
  is extrapolated. It also warns when **Flag Air is 0**: air still clearing
  cal gas lies on a line from the tank toward the atmosphere, which looks like
  a correlation and drags a fit.
- **Linear fit (OLS)** is off by default. These plots often have real
  structure that one straight line describes badly, so the slope would mostly
  describe how the flight was flown.

### Error bars

The bars are the 1σ from the calibration. Write the calibrated value as a
blend of the two assigned values, `c = (1-f)·A_lo + f·A_hi`. The uncertainty
of each term is combined, using for each tank:

| Term | Meaning |
|---|---|
| `sA` | The tank's assigned-value uncertainty (`<GAS>_unc` in `cals.yaml`). Missing counts as 0. |
| `sR` | How well the drift model reproduces that tank's response: the leave-one-out scatter and the closure residual combined. |

Because `f` is not limited to 0–1, the uncertainty grows when the air is
outside the tanks' range. Two things are left out: single-sample instrument
noise, and any inflation over extrapolated spans (those are reported as a
percentage instead). The uncertainties are mostly systematic, which is why the
fit is plain OLS and not weighted by them.

## Export

The Export tab writes two files covering **all gases** in the file. **What
will be exported** shows each gas's status and row count first.

### Companion CSV

Sits **beside** the raw file. It has one row for every raw row, in the same
order, so you can open the two side by side or paste them together with no
alignment. Rows dropped for a stale clock are still there, blank, and marked
in a `presync` column.

| Column | Meaning |
|---|---|
| `<gas>_cal` | The calibrated good-ambient record, with smoothing if set. Blank elsewhere. |
| `<gas>_cal_unc` | 1σ on that value. |
| `<gas>_cal_slope`, `<gas>_cal_intercept` | The coefficients, on every row. |
| `<gas>_pt_factor` | The Pdelt/Tdelt multiplier, when either is on. A blank value can be recomputed as `slope × (raw × pt_factor) + intercept`. |
| `<gas>_is_cal_period`, `_is_post_cal_flush`, `_is_masked`, `_is_extrapolated`, `_is_flagged` | Why a row is blank. |
| `<gas>_filtered`, `<gas>_below_floor` | For O3 and H2O: the value with faults and flags removed. |

Also present: `datetime`, `time_s` (seconds from midnight UTC) and the raw
columns under their original names.

- Masks are 1/0. A blank means "not evaluated".
- Provenance notes go to `<name>_notes.txt`. Tick *Put the provenance notes in
  the CSV as # lines* to embed them.
- A gas with no calibration still exports its raw column and masks.

### ICARTT (.ict)

The archive format (index 1001). It holds the good ambient data only. Every
blanked row is written as the missing value `-99999`, so no mask columns are
needed. A gas with no calibration is left out.

After writing, you are told about rows that were dropped:

- Rows with repeated or backward timestamps, because ICARTT's time must
  strictly increase.
- Rows with no value for any gas (untick *Drop rows with no value for any
  gas* to keep them).

The format follows the sister UCATS instrument's delivered files, so a
campaign's files can be processed the same way: missing is `-99999`, units
carry a trailing `v` (`ppmv`), variable lines read `name, unit,
standard_name, description`, and the 1σ variable is `<species>e_<suffix>`
(`CO2e_RASTA`).

`standard_name` comes from the NASA ESDS Atmospheric Composition naming
convention that ICARTT V2.0 requires. Trace gases are `Gas_CO2_InSitu_S_DMF`
(dry mole fraction). Two differ:

- **H2O** is `Met_H2OMF_InSitu_None`, because water vapour is a `Met`
  quantity in the convention.
- **O3** is `Gas_O3_InSitu_S_AVMR`, a volumetric ratio against ambient air,
  because nothing dries the ozone monitor's sample.

### Header metadata

The form on the Export tab fills in what the header needs that cannot be
derived. Two things are computed for you:

- `UNCERTAINTY` is the median 1σ for each gas. Text you type is appended.
- `ULOD_FLAG`/`LLOD_FLAG` are fixed by the format. Only the LOD values are
  yours (default `N/A`).

**Special comments** and **Revision history** are written verbatim, line
breaks included. The revision history accumulates: an R0 file still lists its
RA line.

`Data ID`, `Location ID` and `Revision` build the suggested name,
`dataID_locationID_YYYYMMDD_R#.ict`. Hyphens are kept (`SABRE-RASTA`);
underscores are stripped. Blank PI, mission, platform or location ID prompts
before writing.

The metadata is saved in `rasta_gui_config.yaml`, not in a flight config,
because PI, affiliation and stipulations belong to the campaign. That file is
tracked in git, so a fresh clone arrives filled in. **Save defaults** writes
it. Export always uses what is in the boxes, saved or not.

## Validity floors

O3 and H2O have no cal bottles. Each has a floor below which a reading is a
sensor fault.

| Gas | Column | Floor |
|---|---|---|
| O3 | `oz_o3best` | −15 ppb |
| H2O | `w_H2Obest` | −5 ppm |

- The floor is well below zero on purpose. Real near-zero ozone scatters
  negative, and clipping at zero would hide the instrument's true noise.
- The floor is not adjustable. It is a physical limit, not a judgement call.
- These gases show two traces: raw in faded blue, and filtered in red. The
  red line breaks over each removed reading, and the note says how many.
  This is always on, whatever the Calibrated combo says.
- The default y-range is framed on the filtered data, so a single −2292 ppb
  fault does not squash the real record. The raw trace runs off-scale.

## Data assumptions

- **Columns.** `datetime`, `j_sol_cals` and `j_sol_aircal`, plus whichever of
  `d1_CO2_ppm`, `d1_N2O_ppb`, `d2_CH4_ppb`, `oz_o3best` and `w_H2Obest` are
  present, and their support columns (`d?_P_mbars`, `d?_T_gas`, `oz_p`, …).
  A missing gas just leaves the selector. A file with none of them errors.
- **Clock metadata.** `sample_id`, `monotonic_ns`, `clock_epoch` and
  `clock_jump_s` are carried through but are not plottable traces.
- **Pre-sync rows.** On startup the logger's clock can be stale for about a
  minute, giving a burst of too-late timestamps and then one large backward
  jump. Everything before the jump is dropped on load.
- **Tank identity.** `j_sol_cals` is a raw valve state, not a tank
  identifier. Each cal window is matched to whichever of the two assigned
  tanks it is closest to, so only the *pair* has to be right.
- **`cals.yaml`** holds every tank's assigned values, plus a `cals:` default
  pair for the current run. It is a local copy of `~/code/ucats-b/cals.yaml`,
  so resync it by hand if that roster changes.

## Files

| File | Purpose |
|---|---|
| `rasta_gui.py` | The PyQt5 viewer. |
| `rasta_analysis.py` | Masking, cal detection, calibration, smoothing and export logic. A library with no GUI dependency, so a script or notebook can reuse it. |
| `rasta_gui_config.yaml` | Shared by git: the [ICARTT header metadata](#header-metadata). No analysis settings. Written only by **Save defaults**. |
| `.rasta_gui_state.yaml` | Local, hidden and gitignored: the recent-files list. |
| `<dataset>_conf.yaml` | Written beside each CSV you save: that flight's settings, cal tanks, flags and smoothing. |
| `cals.yaml` | The cal tank roster and the default pair. |
