# Mechanical Seal Condition Monitoring — Full Technical Code Documentation

This document reverse-engineers the complete MATLAB script that simulates, analyzes, and reports on the health of a mechanical seal on a centrifugal process pump. It explains **why** each line exists, not just what it does, so a reader can understand the entire program without opening MATLAB.

---

# 1. Overall Code Workflow

The script executes as one linear pass, top to bottom. Nothing loops back — every later section consumes the outputs of an earlier one.

```
 Define time vector & phase boundaries
        ↓
 Generate synthetic sensor signals (Temp, Leak, Flush, Pressure)
        ↓
 Define Warning / Alarm / Critical thresholds
        ↓
 Plot time-series with threshold overlays  (Section 4)
        ↓
 Correlate Temperature vs Leakage           (Section 5)
        ↓
 Fit & extrapolate degradation trendlines   (Section 6)
        ↓
 Scan data for threshold violations         (Section 7)
        ↓
 Build combined alarm status dashboard      (Section 8)
        ↓
 Estimate Remaining Useful Life (RUL)       (Section 9)
        ↓
 Print maintenance recommendation
```

Most blocks above map directly onto one of the nine numbered sections in the source `.m` file (Section 1 → time vector, Section 2 → data generation, and so on through Section 9 → RUL estimation). The final block, "Print maintenance recommendation," is not a separate section — it's the last few lines inside Section 9, printed once the RUL calculation completes.

---

# 2. Code Explanation (Piece-by-Piece)

## Section 1 — Time Vector & Dataset Configuration

### Purpose
Establishes the simulation's time base and defines the four operating phases the rest of the script depends on.

### Logic
A real condition-monitoring dataset is a time series. Before any signal can be generated, MATLAB needs a common time axis that every sensor signal will share, plus phase boundary markers so degradation behavior can change at specific points in time.

### MATLAB Code
```matlab
dt          = 1;            % Sampling interval (hours)
t_end       = 1200;         % Total monitoring duration (hours) ~50 days
time        = (0:dt:t_end); % Time vector [hours]
n           = length(time);

t_healthy   = 600;   % Normal operation phase ends at 600 h
t_degrade   = 900;   % Accelerated degradation phase ends at 900 h
t_anomaly   = 1050;  % Sudden anomaly event at 1050 h
```

### Step-by-Step Explanation
- `dt = 1`: the sampling interval — one reading every hour. This keeps the dataset size manageable (1201 points) while still showing clear long-term trends.
- `t_end = 1200`: total simulated monitoring window, roughly 50 days — long enough to show a full healthy-to-failure life cycle.
- `time = (0:dt:t_end)`: builds a row vector `[0, 1, 2, ..., 1200]`. This vector is reused as the x-axis for every plot and every signal calculation in the script.
- `n = length(time)`: the number of samples (1201). This is used repeatedly to size noise vectors and loops.
- `t_healthy`, `t_degrade`, `t_anomaly`: three scalar "checkpoints" that split the 1200-hour timeline into four distinct behavioral regions (explained fully in Section 5, Phase Logic).

---

## Section 2 — Dataset Generation

### Purpose
Creates four synthetic sensor signals — temperature, leakage rate, flush flow rate, pressure differential — that behave like a real degrading mechanical seal.

### Logic
Each signal is built the same conceptual way: **baseline value + phase-dependent drift + random noise + occasional injected events**. This mirrors how real physical degradation looks on a trend chart: a fairly flat signal that slowly drifts, with noise riding on top and the occasional one-off disturbance.

### Index Masks
```matlab
idx_p1 = time <= t_healthy;
idx_p2 = time > t_healthy  & time <= t_degrade;
idx_p3 = time > t_degrade  & time <= t_anomaly;
idx_p4 = time > t_anomaly;
```
`idx_p1`–`idx_p4` are **logical (boolean) vectors** the same length as `time`. Each one is `true` wherever `time` falls inside that phase's window, `false` elsewhere. These masks are used throughout the rest of the script to select and assign values to specific time windows without writing a single loop (see Section 5 for why this matters).

---

### 2.1 Seal Chamber Temperature

**MATLAB Code**
```matlab
temp_base     = 50.0;
temp_noise    = 0.8;
temp_drift_p2 = 0.018;
temp_drift_p3 = 0.045;
temp_step_anom= 14.0;
temp_drift_p4 = 0.06;

seal_chamber_temp = zeros(1, n);

seal_chamber_temp(idx_p1) = temp_base + temp_noise * randn(1, sum(idx_p1));

t2 = time(idx_p2) - t_healthy;
seal_chamber_temp(idx_p2) = temp_base + temp_drift_p2 * t2 + temp_noise * randn(1, sum(idx_p2));

t2_end_val = temp_base + temp_drift_p2 * (t_degrade - t_healthy);
t3 = time(idx_p3) - t_degrade;
seal_chamber_temp(idx_p3) = t2_end_val + temp_drift_p3 * t3 + temp_noise * 1.5 * randn(1, sum(idx_p3));

t3_end_val = t2_end_val + temp_drift_p3 * (t_anomaly - t_degrade);
t4 = time(idx_p4) - t_anomaly;
seal_chamber_temp(idx_p4) = t3_end_val + temp_step_anom + temp_drift_p4 * t4 + temp_noise * 2.0 * randn(1, sum(idx_p4));

spike_times = randsample(n, 18);
seal_chamber_temp(spike_times) = seal_chamber_temp(spike_times) + (2.5 + 3.5 * rand(1, 18)) .* sign(randn(1, 18));
```

**Step-by-Step Explanation**
- `seal_chamber_temp = zeros(1, n)`: **preallocates** the output array. MATLAB arrays that grow inside a loop are slow because MATLAB has to reallocate memory every iteration; preallocating a fixed-size array up front avoids this entirely — a standard MATLAB performance practice.
- **Phase 1 assignment** — `seal_chamber_temp(idx_p1) = ...` uses the logical mask `idx_p1` to write values *only* into the healthy-phase positions of the array, leaving the rest as zero (to be filled in by later lines). This is **logical indexing**: instead of looping `for i = 1:n; if time(i) <= t_healthy ...`, MATLAB applies the operation to the entire masked subset in one vectorized statement.
- `temp_base + temp_noise * randn(1, sum(idx_p1))`: `randn` generates standard normal (Gaussian, mean 0, std 1) random numbers; multiplying by `temp_noise` (0.8 °C) scales the noise to a realistic sensor-noise magnitude, and adding `temp_base` centers it at 50 °C. `sum(idx_p1)` counts how many `true` entries are in the mask — i.e., how many samples fall in Phase 1 — so the noise vector is exactly the right length.
- **Phase 2 (gradual degradation)** — `t2 = time(idx_p2) - t_healthy` re-zeros the time axis so it starts counting from 0 at the beginning of Phase 2. This makes the drift term `temp_drift_p2 * t2` a simple linear ramp starting at the Phase-1 ending temperature, rather than needing to account for the absolute time offset in every term.
- **Phase 3 (accelerated degradation)** — `t2_end_val` calculates exactly what the temperature would be at the end of Phase 2 using the same drift formula, so Phase 3 starts smoothly from where Phase 2 left off (no artificial jump at the boundary). The drift rate more than doubles (`0.045` vs `0.018` °C/h) to represent accelerating wear, and noise amplitude is increased ×1.5 to represent a mechanically less stable condition.
- **Phase 4 (post-anomaly)** — `temp_step_anom = 14.0` is added as an instantaneous jump the moment the anomaly occurs, on top of continued drift (`temp_drift_p4`) and even larger noise (×2.0). This models a real seal-damage event: a sudden step change followed by continued, faster decay.
- **Spike injection** — `randsample(n, 18)` (Statistics and Machine Learning Toolbox) picks 18 random unique indices anywhere in the full 1201-sample array. `(2.5 + 3.5*rand(1,18))` generates spike magnitudes uniformly between 2.5 and 6.0 °C, and `.* sign(randn(1,18))` randomly makes each spike positive or negative. This simulates transient environmental or process disturbances that are not part of the underlying wear trend — real sensor data almost always has a handful of such outliers.

---

### 2.2 Leakage Rate

**MATLAB Code**
```matlab
leak_base     = 0.8;
leak_noise    = 0.15;
leak_drift_p2 = 0.006;
leak_drift_p3 = 0.020;
leak_step_anom= 8.0;
leak_drift_p4 = 0.035;

leakage_rate(idx_p1) = leak_base + leak_noise * abs(randn(1, sum(idx_p1)));
...
leakage_rate = max(leakage_rate, 0.1);
```

**Step-by-Step Explanation**
- The structure mirrors the temperature model exactly (baseline + phase drift + scaled noise + step change at anomaly), but with two important physical corrections:
  - **`abs(randn(...))`** — leakage cannot physically be negative, so the Gaussian noise is folded into its absolute value (a **half-normal distribution**), guaranteeing every noise contribution is positive.
  - **`leakage_rate = max(leakage_rate, 0.1)`** — a final **clamp** applied after all four phases are computed, forcing the absolute floor of the signal to 0.1 ml/min. This protects against any residual negative values and reflects the physical fact that some minimal seepage always exists.
- Drift rates (`0.006` → `0.020` → `0.035` ml/min/h) grow through the phases the same way temperature's does, and the anomaly step (`leak_step_anom = 8.0`) represents a sudden increase in leakage consistent with face damage.

---

### 2.3 Flush Flow Rate

**MATLAB Code**
```matlab
flush_base    = 4.8;
flush_noise   = 0.10;
flush_drift   = -0.0015;

flush_flow_rate = flush_base + flush_drift * time + flush_noise * randn(1, n);

idx_restrict  = time >= 400 & time <= 420;
flush_flow_rate(idx_restrict) = flush_flow_rate(idx_restrict) - 1.2;

flush_flow_rate(idx_p4) = flush_flow_rate(idx_p4) - 0.4;

flush_flow_rate = max(flush_flow_rate, 1.0);
```

**Step-by-Step Explanation**
- Unlike temperature and leakage, flush flow is generated as a **single continuous formula across the entire timeline** (`flush_base + flush_drift * time + noise`) rather than phase-by-phase. `flush_drift = -0.0015` L/min per hour is a very slow, constant *negative* drift representing gradual orifice fouling — this happens independently of the seal's own wear phases.
- `idx_restrict = time >= 400 & time <= 420` is a second logical mask, unrelated to the phase masks, isolating a specific 20-hour window. The `-1.2` subtraction models a temporary flush-line restriction (e.g., a partially clogged strainer) that clears on its own — a realistic, independent process event.
- `flush_flow_rate(idx_p4) = flush_flow_rate(idx_p4) - 0.4` reuses the *same* Phase-4 mask (`idx_p4`) defined back in Section 2's index-mask block, applying a further permanent flow reduction once the seal anomaly occurs (consistent with debris/damage partially obstructing flow).
- `max(flush_flow_rate, 1.0)` clamps the signal to a physically sensible lower bound.

---

### 2.4 Pressure Differential

**MATLAB Code**
```matlab
press_base    = 2.0;
press_noise   = 0.08;
press_drift   = 0.0008;

pressure_differential = press_base + press_drift * time + press_noise * randn(1, n);

pressure_differential(idx_p4) = pressure_differential(idx_p4) + 0.45 - 0.0002 * (time(idx_p4) - t_anomaly);

idx_spike_p = time >= 750 & time <= 760;
pressure_differential(idx_spike_p) = pressure_differential(idx_spike_p) + 0.7;

pressure_differential = max(pressure_differential, 0.2);
```

**Step-by-Step Explanation**
- Like flush flow, pressure is built as one continuous formula with a very slow, constant positive drift (`0.0008` bar/h), representing gradual system pressure build-up unrelated to seal phase.
- At the anomaly (`idx_p4`), pressure jumps up by 0.45 bar, then *slowly relaxes* via the negative term `-0.0002 * (time - t_anomaly)` — modeling a pressure spike that partially bleeds off over time, rather than staying permanently elevated.
- `idx_spike_p` is a third independent event mask (750–760 h), adding a short-lived 0.7 bar transient spike to represent a one-off process upset, unrelated to the underlying degradation trend.
- Final `max(..., 0.2)` clamp again enforces a physical lower bound.

---

## Section 3 — Threshold Definitions

### Purpose
Encodes the alarm philosophy for the whole monitoring system in one place.

### Logic
Rather than scattering numeric limits throughout the code, all thresholds are collected into a single MATLAB **struct**, `thresholds`, so every later section (plotting, alarm detection, dashboard, RUL) references the same source of truth.

### MATLAB Code
```matlab
thresholds = struct();

thresholds.temp_warn     = 65;
thresholds.temp_alarm    = 72;
thresholds.temp_critical = 80;

thresholds.leak_warn     = 4.0;
thresholds.leak_alarm    = 6.0;
thresholds.leak_critical = 12.0;

thresholds.flush_low_warn   = 3.8;
thresholds.flush_low_alarm  = 3.0;
thresholds.flush_high_warn  = 6.0;

thresholds.press_high_warn  = 3.2;
thresholds.press_high_alarm = 3.8;
thresholds.press_low_alarm  = 0.8;
```

### Step-by-Step Explanation
- `struct()` creates an empty structure; each `thresholds.<field> = value` line adds a named field. Using a struct (instead of separate variables like `tempWarn`, `tempAlarm`, etc.) keeps related values grouped and makes the code self-documenting — `thresholds.temp_alarm` reads clearly wherever it's used later.
- Flush flow and pressure each have **two-sided** limits (a low-side and a high-side warning/alarm), because both too little and too much flow/pressure indicate abnormal conditions — unlike temperature and leakage, which are only concerning when too high.

---

## Section 4 — Time-Series Plots

### Purpose
Visualizes all four raw signals together on a shared time axis so a reviewer can see the whole seal's health at a glance.

### Logic
A `4×1` tiled layout stacks Temperature, Leakage, Flush Flow, and Pressure vertically, sharing the same x-axis, so events that occur in one signal (like the anomaly at 1050 h) can be visually cross-referenced against the others in the same instant.

### MATLAB Code (representative excerpt)
```matlab
tl = tiledlayout(4,1,'TileSpacing','compact','Padding','compact');

ax1 = nexttile;
plot(time, seal_chamber_temp, 'Color', cTemp, 'LineWidth', 1.2);
yline(thresholds.temp_warn, '--', 'Color', warnColor);
yline(thresholds.temp_alarm, '--', 'Color', alarmColor);
yline(thresholds.temp_critical, '-.', 'Color', criticalColor, 'LineWidth', 1.3);
xline(t_anomaly, 'r--', 'LineWidth', 1.3);
...
linkaxes([ax1 ax2 ax3 ax4], 'x');
```

### Step-by-Step Explanation
- `tiledlayout(4,1,...)` creates a 4-row, 1-column grid of plot axes with `'TileSpacing','compact'` to minimize wasted space between subplots — better for readability than MATLAB's older `subplot` spacing defaults.
- Each `nexttile` call moves to the next tile in the grid and returns an axis handle (`ax1`–`ax4`), which is stored specifically so the axes can be linked afterward.
- `yline(...)` draws a horizontal reference line at a fixed y-value — used here to overlay the Warning, Alarm, and Critical thresholds directly on top of each signal, so a reader can instantly see how close the trend is to breaching a limit.
- `xline(t_anomaly, 'r--', ...)` draws a vertical dashed red line at hour 1050 on every subplot, marking the moment of the sudden anomaly event across all four parameters simultaneously.
- `linkaxes([ax1 ax2 ax3 ax4], 'x')` locks the x-axis of all four subplots together — zooming or panning one automatically applies to the rest, which is essential for comparing events across signals.
- A manual legend is built afterward using invisible dummy plots (`plot(NaN,NaN,...)`) so the Warning/Alarm/Critical line styles appear in a single shared legend at the top of the figure rather than being repeated four times.

---

## Section 5 — Correlation Analysis (Temperature vs. Leakage)

### Purpose
Quantifies whether temperature and leakage rise together, providing statistical evidence that the two failure indicators are physically linked.

### MATLAB Code
```matlab
[r_pearson, p_value] = corrcoef(seal_chamber_temp, leakage_rate);
r_val = r_pearson(1,2);
p_val = p_value(1,2);

p_fit = polyfit(seal_chamber_temp, leakage_rate, 1);
x_fit = linspace(min(seal_chamber_temp), max(seal_chamber_temp), 300);
y_fit = polyval(p_fit, x_fit);
```

### Step-by-Step Explanation
- `corrcoef` returns a 2×2 correlation matrix; the diagonal entries are always 1 (a signal is perfectly correlated with itself), so the meaningful value is the off-diagonal entry `r_pearson(1,2)` — the correlation between temperature and leakage specifically.
- `polyfit(x, y, 1)` fits a first-degree (linear) polynomial — i.e., a straight line `y = p(1)*x + p(2)` — through the scatter of temperature vs. leakage points using least-squares regression.
- `linspace(min(...), max(...), 300)` generates 300 evenly spaced x-values spanning the observed temperature range, purely so the fitted line can be drawn smoothly across the plot; `polyval` then evaluates the fitted polynomial at each of those 300 points.
- The result, **r = 0.990**, is interpreted in-code with a simple threshold rule (`abs(r_val) > 0.8` → "Strong positive correlation"), giving an automatic, readable classification rather than requiring the user to interpret the raw number themselves.

---

## Section 6 — Trendline Fitting & Extrapolation

### Purpose
Projects the leakage and temperature trends forward in time to visualize where they are heading if degradation continues unchecked.

### MATLAB Code
```matlab
p_leak_lin  = polyfit(time, leakage_rate, 1);
p_leak_poly = polyfit(time, leakage_rate, 3);

t_extrap = time(end) : dt : time(end) + 300;
leak_extrap = polyval(p_leak_poly, t_extrap);
```
*(identical structure repeated for temperature)*

### Step-by-Step Explanation
- Two models are fit to the **same data**: a straight line (`polyfit(..., 1)`) and a cubic polynomial (`polyfit(..., 3)`). The linear fit shows the *average* rate of change across the whole 1200 hours, while the cubic fit can bend to follow the accelerating (non-linear) shape of real degradation.
- `t_extrap = time(end) : dt : time(end) + 300` builds a new time vector picking up exactly where the measured data ends and continuing 300 hours further, at the same 1-hour resolution.
- `polyval(p_leak_poly, t_extrap)` evaluates the fitted cubic curve *beyond* the range it was fit to — this is **extrapolation**, and it is inherently less reliable than interpolation (evaluating within the fitted range), which is why the extrapolated segment is drawn with a dotted line style in the plot to visually flag it as a projection rather than a confirmed measurement.
- The reported linear slopes (0.0104 ml/min/h for leakage, 0.0215 °C/h for temperature) describe the *average* degradation rate — useful for a simple linear runway estimate, but they understate how fast the seal degrades in the final 150 hours, which is exactly why Section 9 uses the cubic fit (not the linear one) for the actual RUL calculation.

---

## Section 7 — Threshold-Based Alarm Detection

### Purpose
Automatically scans the full dataset and reports every instance where a signal crossed a defined limit — the core function of any real alarm-management system.

### MATLAB Code
```matlab
printAlarm = @(idx,label,limit,unit) ...
    fprintf('  %-10s (%s %.1f %s) : %4d samples | First = %5.0f h | Last = %5.0f h\n',...
    label, limit.sign, limit.value, unit, numel(idx), time(idx(1)), time(idx(end)));

alarms = {
    find(seal_chamber_temp > thresholds.temp_warn),     'WARNING',  struct('sign','>','value',thresholds.temp_warn);
    find(seal_chamber_temp > thresholds.temp_alarm),    'ALARM',    struct('sign','>','value',thresholds.temp_alarm);
    find(seal_chamber_temp > thresholds.temp_critical), 'CRITICAL', struct('sign','>','value',thresholds.temp_critical);
};

for k = 1:size(alarms,1)
    idx = alarms{k,1};
    if ~isempty(idx)
        printAlarm(idx, alarms{k,2}, alarms{k,3}, '°C');
    end
end
```

### Step-by-Step Explanation
- `printAlarm = @(...) fprintf(...)` defines an **anonymous function** — a small, reusable formatting routine — so the same print logic doesn't have to be rewritten for every parameter and every severity level.
- `find(seal_chamber_temp > thresholds.temp_warn)` returns the **indices** (not the values) where the condition is true — i.e., every sample number where temperature exceeded the warning threshold.
- The `alarms` variable is a **cell array**: each row bundles together the matching indices, a text label, and a small struct describing the comparison sign and threshold value. This lets the `for` loop iterate through all three severity levels (Warning, Alarm, Critical) generically, without writing three nearly-identical blocks of code.
- Inside the loop, `if ~isempty(idx)` skips printing anything for a severity level that was never reached (e.g., "Critical" for flush flow, which only has low-side limits defined) — keeping the printed report clean.
- `numel(idx)` counts how many samples breached the threshold; `time(idx(1))` and `time(idx(end))` look up the *first* and *last* times (in hours) that the breach occurred — because `find` always returns indices in ascending order, `idx(1)` is guaranteed to be the earliest breach and `idx(end)` the latest.
- This same pattern (define comparison → `find` → loop → `printAlarm`) is repeated for Leakage, Flush Flow, and Pressure, with the sign flipped to `<` where a *low* value is the abnormal condition (e.g., low flush flow).

---

## Section 8 — Combined Alarm Status Dashboard

### Purpose
Converts each parameter's alarm state, at every point in time, into a single numeric severity code, so all four signals can be visualized together as one color-coded image.

### MATLAB Code
```matlab
idx_temp_warn     = seal_chamber_temp > thresholds.temp_warn;
idx_temp_alarm    = seal_chamber_temp > thresholds.temp_alarm;
idx_temp_critical = seal_chamber_temp > thresholds.temp_critical;

alarm_temp = zeros(1,n);
alarm_temp(idx_temp_warn)     = 1;
alarm_temp(idx_temp_alarm)    = 2;
alarm_temp(idx_temp_critical) = 3;

alarm_matrix = [alarm_temp; alarm_leak; alarm_flush; alarm_press];

imagesc(time, 1:4, alarm_matrix);
colormap(alarm_cmap);
clim([-0.5 3.5]);
```

### Step-by-Step Explanation
- Three logical masks are created per parameter, one for each severity level. Because a "Critical" sample is *also* true for "Warning" and "Alarm" (72 °C is greater than both 65 °C and 72 °C's own threshold, since `>` is inclusive of higher values), the assignments are deliberately made **in order from lowest to highest severity** (`1`, then `2`, then `3`) — so a sample that qualifies for all three ends up correctly overwritten with the highest applicable code, `3`.
- Each parameter is condensed into a single **severity-coded row vector** (0 = OK, 1 = Warning, 2 = Alarm, 3 = Critical), then all four rows are stacked into one `4×n` matrix, `alarm_matrix`, with `vertcat` (the `[...]` syntax).
- `imagesc(time, 1:4, alarm_matrix)` renders this matrix directly as a color-mapped image: the x-axis is time, the y-axis is the row number (1 through 4, one per parameter), and each pixel's color represents its severity code.
- `colormap(alarm_cmap)` applies a custom 4-color lookup table (green → yellow → orange → red) so severity codes 0–3 map to intuitive traffic-light colors.
- `clim([-0.5 3.5])` — short for "color limits" — forces the color axis to treat the data as **exactly four discrete bins** centered on 0, 1, 2, 3, rather than letting MATLAB auto-scale to a continuous gradient, which would blur the boundaries between severity levels.
- `xline` calls mark the phase boundaries (`t_healthy`, `t_degrade`, `t_anomaly`) directly on the dashboard so the reader can see which operating phase corresponds to which alarm pattern.

---

## Section 9 — Remaining Useful Life (RUL) Estimation

### Purpose
Simulates what a real-time monitoring engineer would see: using only the data available up to a given moment, predict when the seal will fail, then check that prediction against what actually happened.

### MATLAB Code
```matlab
prediction_times = [900 1050];
polyOrder = 3;
tol = 1e-8;

actualLeakFailure = time(find(leakage_rate >= thresholds.leak_critical, 1, 'first'));
actualTempFailure = time(find(seal_chamber_temp >= thresholds.temp_critical, 1, 'first'));

for k = 1:length(prediction_times)
    prediction_time = prediction_times(k);

    idx_fit  = (time >= t_healthy) & (time <= prediction_time);
    time_fit = time(idx_fit);
    leak_fit = leakage_rate(idx_fit);

    pLeak = polyfit(time_fit, leak_fit, polyOrder);

    coeff = pLeak;
    coeff(end) = coeff(end) - thresholds.leak_critical;
    r = roots(coeff);

    valid = real(r(abs(imag(r)) < tol & real(r) > prediction_time));

    if isempty(valid)
        tFailLeak = NaN;
        RULLeak = NaN;
    else
        tFailLeak = min(valid);
        RULLeak = tFailLeak - prediction_time;
    end
    ...
    overallRUL = min([RULLeak RULTemp], [], 'omitnan');
end
```

### Step-by-Step Explanation
- `prediction_times = [900 1050]` defines two "snapshot" moments at which the script pretends it doesn't yet know the future — this recreates a realistic scenario where an engineer is monitoring live data and has no visibility past the current hour.
- `find(leakage_rate >= thresholds.leak_critical, 1, 'first')` finds the **first** sample where leakage actually reaches the critical threshold across the *entire* dataset — this is only possible because the script has the complete simulated future available for **validation purposes**; a real system wouldn't have this until it happened.
- Inside the loop: `idx_fit = (time >= t_healthy) & (time <= prediction_time)` masks the data to **only** the window from the end of the healthy phase up to the current prediction time — deliberately excluding both the flat healthy baseline (which would bias the polynomial fit) and anything after the prediction point (which would be "cheating" by using future data).
- `polyfit(time_fit, leak_fit, polyOrder)` fits a cubic polynomial to only that visible window of data, producing coefficients `pLeak` describing a curve `p(1)*t^3 + p(2)*t^2 + p(3)*t + p(4)`.
- **Finding when the curve crosses the critical threshold** is done by root-finding rather than a search loop:
  - `coeff(end) = coeff(end) - thresholds.leak_critical` shifts the polynomial down by the threshold value. This turns the question "when does `p(t) = 12`?" into the equivalent, solvable question "when does `p(t) - 12 = 0`?"
  - `roots(coeff)` finds all values of `t` where this shifted polynomial equals zero — a cubic has up to three roots, which may be real or complex.
  - `abs(imag(r)) < tol` filters out complex roots (keeping only those whose imaginary part is essentially zero, allowing for floating-point rounding via the tolerance `tol`), since only real-valued times are physically meaningful.
  - `real(r) > prediction_time` keeps only roots that occur *after* the current prediction time — a mathematically valid root in the past would represent an already-known historical crossing, not a future prediction.
  - `min(valid)` — if multiple valid future roots exist, the **earliest** one is the first time the curve is predicted to cross the threshold, so `tFailLeak` takes the smallest surviving value.
  - If no valid root remains (`isempty(valid)`), the polynomial simply never re-crosses the threshold within a real, future timeframe based on the data seen so far — this is exactly what happens at the 900-hour prediction, because not enough of the accelerating trend has appeared yet for the cubic fit to curve upward convincingly.
- `RULLeak = tFailLeak - prediction_time` converts the *absolute* failure time into a **remaining** duration relative to "now" (the prediction time).
- The identical process is run for temperature, producing `RULTemp`.
- `overallRUL = min([RULLeak RULTemp], [], 'omitnan')` takes the more conservative (shorter) of the two RUL estimates as the overall answer — the seal is considered at end-of-life the moment *either* parameter is predicted to fail, and `'omitnan'` ensures that if one parameter's RUL couldn't be calculated, the valid one is still used rather than the whole result collapsing to `NaN`.
- A simple decision rule (`if overallRUL <= 200 → "Schedule maintenance immediately"`) turns the numeric RUL into an actionable maintenance recommendation.

---

# 3. Mathematical Models

### Temperature Model

The signal is built from four separate equations, one per operating phase.
 
**Phase 1 — Healthy ($t \le t_h$):**
 
$$
T(t) = T_0 + \varepsilon(t)
$$
 
**Phase 2 — Gradual degradation ($t_h < t \le t_d$):**
 
$$
T(t) = T_0 + k_2(t - t_h) + 1.0 \, \varepsilon(t)
$$
 
**Phase 3 — Accelerated degradation ($t_d < t \le t_a$):**
 
$$
T(t) = T_d + k_3(t - t_d) + 1.5 \, \varepsilon(t)
$$
 
**Phase 4 — Post-anomaly ($t > t_a$):**
 
$$
T(t) = T_a + \Delta T_{step} + k_4(t - t_a) + 2.0 \, \varepsilon(t)
$$

Where $T_0 = 50\,°C$ is the baseline, $t_h, t_d, t_a$ are the healthy/degradation/anomaly boundary times (600, 900, 1050 h), $k_2, k_3, k_4$ are the phase drift rates (0.018, 0.045, 0.06 °C/h), $\Delta T_{step} = 14\,°C$ is the anomaly step, $T_d$ and $T_a$ are the carried-forward values from the end of the previous phase, and $\varepsilon(t) \sim \mathcal{N}(0, 0.8^2)$ is Gaussian sensor noise.

### Leakage Model

$$
L(t) = \max\Big(0.1,\; L_0 + k_L(t)\cdot\Delta t + |\varepsilon_L(t)|\Big)
$$

Where $L_0 = 0.8$ ml/min is the baseline, $k_L(t)$ is the phase-dependent drift rate (0.006 → 0.020 → 0.035 ml/min/h, plus an 8 ml/min step at the anomaly), and $|\varepsilon_L(t)|$ is half-normal noise (the absolute value of a Gaussian), enforcing non-negativity.

### Flush Flow Model

$$
F(t) = \max\big(1.0,\; F_0 + k_F \cdot t + \varepsilon_F(t) - R(t) - S\cdot\mathbb{1}[t > t_a]\big)
$$

Where $F_0 = 4.8$ L/min, $k_F = -0.0015$ L/min/h is the slow fouling drift, $R(t) = 1.2$ during the 400–420 h restriction window (else 0), and $S = 0.4$ L/min is the permanent post-anomaly flow reduction.

### Pressure Model

$$
P(t) = \max\Big(0.2,\; P_0 + k_P \cdot t + \varepsilon_P(t) + \big[0.45 - 0.0002(t-t_a)\big]\mathbb{1}[t > t_a] + 0.7\cdot\mathbb{1}[750 \le t \le 760]\Big)
$$

Where $P_0 = 2.0$ bar, $k_P = 0.0008$ bar/h.

### Linear Regression (Correlation & Trend Fit)

$$
y = m x + b
$$

Fit by least-squares via `polyfit(x, y, 1)`, minimizing $\sum_i (y_i - (mx_i+b))^2$.

### Pearson Correlation Coefficient

$$
r = \frac{\sum_i (x_i - \bar{x})(y_i - \bar{y})}{\sqrt{\sum_i (x_i-\bar{x})^2}\sqrt{\sum_i(y_i-\bar{y})^2}}
$$

### Cubic Trend/Extrapolation Model

$$
y(t) = a_3 t^3 + a_2 t^2 + a_1 t + a_0
$$

Fit via `polyfit(t, y, 3)` on measured data, then evaluated (`polyval`) beyond the measured range for extrapolation.

### RUL Root-Finding Equation
Given a fitted cubic $p(t)$ and critical threshold $C$, MATLAB solves for the roots of the shifted polynomial:
 
$$
p(t) - C = 0
$$
 
Each root $t$ is checked against two conditions: it must be real (negligible imaginary part), and it must lie in the future relative to the prediction time $t_{pred}$ (i.e. $t > t_{pred}$). The predicted failure time $t_f$ is the smallest root that satisfies both conditions. The Remaining Useful Life is then:
 
$$
RUL = t_f - t_{pred}
$$
 
---

# 4. Phase Logic

The 1200-hour timeline is split into four operating phases using **logical masks**:

```matlab
idx_p1 = time <= t_healthy;                       % 0–600 h:   Healthy
idx_p2 = time > t_healthy  & time <= t_degrade;    % 600–900 h: Gradual degradation
idx_p3 = time > t_degrade  & time <= t_anomaly;    % 900–1050h: Accelerated degradation
idx_p4 = time > t_anomaly;                         % 1050–1200h: Post-anomaly/failure
```

| Phase | Window | Behavior |
|---|---|---|
| Healthy | 0–600 h | Flat baseline, mild noise |
| Gradual Degradation | 600–900 h | Slow linear drift begins |
| Accelerated Degradation | 900–1050 h | Faster drift, more noise |
| Post-Anomaly | 1050–1200 h | Step change + fastest decay |

**Why logical masks instead of loops:** A mask like `idx_p2` is a vector of `true`/`false` values the same length as `time`. Using it as an index (`signal(idx_p2) = ...`) tells MATLAB to operate on every matching element **simultaneously**, using MATLAB's underlying vectorized (compiled, array-oriented) operations. A `for` loop checking `if time(i) > t_healthy && time(i) <= t_degrade` for each of the 1201 samples would produce an identical result but run slower and require more code, since MATLAB is optimized for whole-array operations rather than element-by-element iteration.

---

# 5. Sensor Models

| Sensor | Engineering Meaning | Noise Type | Physical Constraint | Industrial Interpretation |
|---|---|---|---|---|
| Seal Chamber Temperature | Heat generated by face friction inside the seal chamber | Gaussian, scaled up per phase | None explicit (spikes are additive) | Rising temperature = declining lubrication/film integrity |
| Leakage Rate | Fluid escaping past the seal faces | Half-normal (`abs(randn)`) | Floor of 0.1 ml/min | Direct indicator of face wear and seal integrity |
| Flush Flow Rate | Cooling/lubricating flow per API Plan 11 | Gaussian | Floor of 1.0 L/min | Falling flow → reduced cooling, accelerates wear |
| Pressure Differential | Pressure across the seal faces | Gaussian | Floor of 0.2 bar | Spikes indicate process upsets; sustained rise may indicate mechanical distress |

Each sensor's **expected behavior** is: flat and noisy while healthy, then increasingly drifting/unstable as the simulation progresses through Phases 2–4 — deliberately designed so that a plot of any one signal alone tells a partial story, but all four together confirm a consistent physical narrative (as demonstrated by the temperature–leakage correlation in Section 5 of the code).

---

# 6. Threshold Logic

| Parameter | Warning | Alarm | Critical |
|---|---|---|---|
| Temperature | > 65 °C | > 72 °C | > 80 °C |
| Leakage | > 4.0 ml/min | > 6.0 ml/min | > 12.0 ml/min |
| Flush Flow (low) | < 3.8 L/min | < 3.0 L/min | — |
| Flush Flow (high) | > 6.0 L/min | — | — |
| Pressure (high) | > 3.2 bar | > 3.8 bar | — |
| Pressure (low) | — | < 0.8 bar | — |

These three-tier limits (Warning → Alarm → Critical) reflect standard industrial alarm philosophy: **Warning** flags an early deviation worth watching, **Alarm** requires operator attention, and **Critical** typically triggers an automatic shutdown or emergency response recommendation. Flush flow and pressure use **two-sided** limits because deviation in either direction (too little or too much) is abnormal, whereas temperature and leakage are one-sided (only "too high" is a problem).

In MATLAB, each comparison (`seal_chamber_temp > thresholds.temp_warn`) produces a logical array; assigning increasing integer codes (1, 2, 3) for Warning, Alarm, Critical — applied in that specific order — lets a single numeric array represent a sample's alarm state, which is what makes the Section 8 dashboard possible.

---

# 7. Statistical Analysis

| Analysis | MATLAB Function | Purpose |
|---|---|---|
| Pearson Correlation | `corrcoef` | Quantify the strength/direction of the temperature–leakage relationship |
| Linear Regression | `polyfit(x,y,1)` | Fit a straight-line relationship for the correlation plot and for average degradation rate |
| Polynomial Trend Fitting | `polyfit(t,y,3)` | Capture accelerating (non-linear) degradation shape |
| Extrapolation | `polyval` beyond fitted range | Project the fitted curve forward in time |
| Root-Finding | `roots` | Solve for the time at which a fitted curve crosses a threshold (used in RUL) |

No moving-average smoothing or dedicated forecasting toolbox functions are used — trend fitting relies entirely on `polyfit`/`polyval`/`roots`, which are base-MATLAB numerical functions.

---

# 8. Visualization Logic

| Figure | Purpose | Axes | Industrial Significance |
|---|---|---|---|
| Time-Series (Sec. 4) | Show raw trends for all 4 parameters together | X: time (h), Y: each parameter's units | Lets an engineer visually correlate events across signals at a glance |
| Correlation Scatter (Sec. 5) | Quantify and visualize the Temp–Leak relationship, colored by phase | X: Temp (°C), Y: Leak (ml/min) | Confirms two independent sensors are telling the same physical story |
| Trendline & Extrapolation (Sec. 6) | Compare linear vs. polynomial fits and project forward | X: time (h), Y: parameter value | Shows why a simple linear "runway" estimate underestimates real accelerating wear |
| Alarm Dashboard (Sec. 8) | Single glance health summary across all parameters and all time | X: time (h), Y: parameter (categorical) | Mimics a SCADA/HMI alarm summary screen used on a real control room display |

---

# 9. Engineering Assumptions

- **Synthetic, not measured, data.** Every value is generated by formula plus randomness, not sampled from a real installation.
- **Discrete four-phase structure.** Real wear is continuous; the phase boundaries (600 h, 900 h, 1050 h) are simplifications for clarity and teaching value.
- **Noise models are simplified.** Gaussian noise (temperature, flush, pressure) and half-normal noise (leakage) approximate — but don't fully replicate — real sensor noise, which can include drift, quantization steps, or non-stationary variance.
- **Fixed thresholds, not adaptive limits.** Warning/Alarm/Critical values are static constants, not derived from statistical process control or manufacturer-specific data sheets.
- **One failure mode only.** The model represents progressive face wear plus a single sudden anomaly — not the many other real seal failure modes (dry running, chemical attack, thermal shock, misalignment, improper installation).
- **No sensor faults simulated.** All four sensors are assumed to report correctly for the entire simulation; no dropout, drift, or miscalibration is modeled.
- **Independent transient events.** The flush restriction (400–420 h) and pressure spike (750–760 h) are injected as standalone events unrelated to the underlying degradation curve.
- **1-hour sampling resolution.** Real plant historians often sample every few seconds to minutes; this project intentionally uses a coarser 1-hour interval to keep dataset size and plots manageable.
- **Polynomial RUL model, not physics-based.** RUL is derived from curve-fitting the *symptoms* (temperature, leakage), not from a mechanistic wear equation (e.g., Archard's wear law) or a trained machine-learning prognostics model.

---

# 10. Code Design Decisions

| Choice | Why chosen (over the alternative) |
|---|---|
| **Logical indexing** (`signal(mask) = ...`) | Faster and more concise than `for`/`if` loops over 1201 samples; leverages MATLAB's native vectorized array operations |
| **Preallocation** (`zeros(1,n)`) | Avoids the performance penalty of an array growing one element at a time inside a loop |
| **Polynomial fitting** (`polyfit`/`polyval`) | Simple, built-in, and interpretable — sufficient to demonstrate trend-based RUL without requiring a dedicated toolbox or trained model |
| **Threshold monitoring** (fixed comparisons) | Matches how real plant alarm systems are actually configured (fixed setpoints), making the demonstration directly transferable to industrial practice |
| **Random noise (`randn`/`rand`)** | Produces a dataset that looks and behaves like real sensor data, rather than an unrealistically "clean" mathematical curve |
| **Synthetic data generation** | Removes the need for access to proprietary or hard-to-obtain plant data, while still producing a dataset with known, verifiable "ground truth" failure times — which is exactly what makes the RUL validation in Section 9 possible |
| **Struct for thresholds** (`thresholds.temp_warn`, etc.) | Groups related constants under one readable, self-documenting variable instead of many loose numeric variables |
| **Cell array for alarm definitions** (`alarms = {...}`) | Lets one generic loop process Warning/Alarm/Critical for each parameter, instead of duplicating near-identical code blocks three times per parameter |

---

## Summary

This script is a compact but complete demonstration of the reliability-engineering workflow used in real predictive maintenance systems: generate/receive sensor data → visualize it → quantify relationships between parameters → fit and extrapolate degradation trends → monitor against fixed thresholds → summarize alarm state visually → estimate remaining useful life → issue a maintenance recommendation. Every mathematical and coding choice mirrors a standard, real-world equivalent, making this a practical reference for anyone building similar condition-monitoring logic on live plant data.
