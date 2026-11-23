# Lesson 012 — MATLAB: detect telemetry anomalies with a causal moving-median baseline

**Date:** 2026-11-23  
**Domain:** MATLAB / numerical analysis / observability  
**Duration:** 30 minutes  
**Environment:** MATLAB R2019a or newer (recommended: a currently supported release); base MATLAB only, no additional toolbox required.  
**Version notes:** `readmatrix` and `writematrix` are available from R2019a. The exercise uses `timetable` and `isoutlier` with a trailing moving-median window. The script is intended for MATLAB, not GNU Octave's API subset.

## Objective

Build a small, reproducible telemetry-analysis job that imports a CSV, validates the data, detects latency/CPU/error-rate spikes using a causal moving-median (Hampel-style) detector, and exports a machine-readable anomaly report.

Practise:

1. Numeric CSV import/export with `readmatrix` and `writematrix`.
2. Time-indexed metrics with a MATLAB `timetable`.
3. A trailing moving-median outlier detector using `isoutlier`.
4. Basic data-quality gates before analysis.
5. A separate report artifact and a plot for operator review.

**Safety:** The dataset is generated locally and contains synthetic values. No production telemetry, endpoints, customer data or external services are used.

## Timed plan

| Time | Activity |
|---|---|
| 0–4 min | Check MATLAB release and required functions |
| 4–9 min | Generate deterministic synthetic telemetry and CSV |
| 9–14 min | Import, validate and build a timetable |
| 14–22 min | Detect anomalies with a causal rolling baseline |
| 22–26 min | Export the anomaly report and plot the results |
| 26–30 min | Review operational caveats and self-check |

## Official documentation

1. [MathWorks — isoutlier](https://www.mathworks.com/help/matlab/ref/isoutlier.html) — moving-median method, window semantics and scaled MAD.
2. [MathWorks — readmatrix](https://www.mathworks.com/help/matlab/ref/readmatrix.html) — import numeric data from CSV and other files.

The `movmedian` method uses a local median and scaled median absolute deviation (MAD). For a two-element window `[b f]`, MATLAB uses `b` observations before and `f` observations after the current sample. The exercise uses `[14 0]` to avoid looking into the future.

## Practical exercise

### 1. Preflight — 4 minutes

Open MATLAB. In the Command Window run:

```matlab
version('-release')
which readmatrix
which writematrix
which timetable
which isoutlier
```

Expected:
- MATLAB release R2019a or newer.
- All four functions resolve to MATLAB paths.
- No toolbox installation is required for this exercise.

If MATLAB is not installed, do not install or purchase it just for this lesson. Record the preflight as blocked and keep the exercise for an existing licensed MATLAB environment.

### 2. Create the lab script — 5 minutes

In MATLAB, create a working folder under the current folder:

```matlab
labDir = fullfile(pwd, 'lesson-012-matlab-telemetry');
if ~isfolder(labDir)
    mkdir(labDir);
end
edit(fullfile(labDir, 'telemetry_anomaly_lab.m'))
```

Paste the complete script below into the editor and save it.

```matlab
clear;
clc;
close all;

% Keep the lab deterministic so the expected anomalies are repeatable.
rng(42, 'twister');

labDir = fileparts(mfilename('fullpath'));
if isempty(labDir)
    labDir = pwd;
end

csvPath = fullfile(labDir, 'synthetic-telemetry.csv');
reportPath = fullfile(labDir, 'anomaly-report.csv');

% 120 one-minute samples of synthetic service telemetry.
n = 120;
sample = (1:n)';

cpuPct = 45 + 4*sin(sample/12) + 0.4*randn(n,1);
p95Ms = 180 + 15*sin(sample/10) + 1.5*randn(n,1);
errorPct = max(0, 0.25 + 0.02*randn(n,1));

% Inject three known latency spikes and one correlated CPU/error spike.
p95Ms([40 85 100]) = [1200; 950; 1400];
cpuPct(85) = 96;
errorPct(85) = 8;

raw = [sample cpuPct p95Ms errorPct];
writematrix(raw, csvPath);

% Read the CSV back: analyse the persisted artifact, not just workspace data.
data = readmatrix(csvPath);

% Data-quality gates before running the detector.
assert(size(data,1) == n, 'Unexpected row count in telemetry CSV.');
assert(size(data,2) == 4, 'Expected four numeric columns.');
assert(all(isfinite(data(:))), 'CSV contains NaN or Inf values.');
assert(all(diff(data(:,1)) == 1), 'Sample sequence has a gap or duplicate.');
assert(all(data(:,2) >= 0 & data(:,2) <= 100), ...
    'CPU percentage is outside 0..100.');
assert(all(data(:,3) >= 0), 'Latency contains a negative value.');
assert(all(data(:,4) >= 0), 'Error rate contains a negative value.');

% Reconstruct one-minute UTC sample times and create a timetable.
time = datetime(2026,11,23,12,0,0,'TimeZone','UTC') ...
       + minutes(data(:,1)-1);

TT = timetable(time, data(:,2), data(:,3), data(:,4), ...
    'VariableNames', {'CpuPct','P95Ms','ErrorPct'});

% 14 previous samples + current sample; zero future samples.
% This is a causal window suitable for an online-style check.
window = [14 0];

cpuOutlier = isoutlier(TT.CpuPct, 'movmedian', window);
p95Outlier = isoutlier(TT.P95Ms, 'movmedian', window);
errorOutlier = isoutlier(TT.ErrorPct, 'movmedian', window);

anyAlert = cpuOutlier | p95Outlier | errorOutlier;

% Export a separate report; do not overwrite the source telemetry.
report = table(time, cpuOutlier, p95Outlier, errorOutlier, anyAlert, ...
    'VariableNames', {'TimestampUtc','CpuOutlier','P95Outlier', ...
                      'ErrorOutlier','AnyAlert'});
writetable(report, reportPath);

alertSamples = data(anyAlert, 1);

fprintf('Samples analysed: %d\n', height(TT));
fprintf('Samples flagged:  %d\n', nnz(anyAlert));
fprintf('Flagged sample numbers: ');
fprintf('%d ', alertSamples);
fprintf('\nCSV:    %s\nReport: %s\n', csvPath, reportPath);

% Plot p95 latency and mark detected points.
figure('Name', 'Lesson 012 - telemetry anomaly review');
plot(time, TT.P95Ms, '-');
hold on;
plot(time(p95Outlier), TT.P95Ms(p95Outlier), 'ro', ...
    'MarkerFaceColor', 'r');
grid on;
ylabel('p95 latency (ms)');
xlabel('UTC time');
title('p95 latency: trailing moving-median outliers');
legend('p95 latency', 'flagged samples', 'Location', 'best');
```

### 3. Run and validate the data — 5 minutes

In the MATLAB Command Window:

```matlab
run(fullfile(labDir, 'telemetry_anomaly_lab.m'))
```

Expected:
- 120 samples analysed.
- The injected p95 spikes at samples 40, 85 and 100 should be flagged.
- Sample 85 should also show CPU and error-rate anomalies.
- The script creates `synthetic-telemetry.csv`, `anomaly-report.csv` and a plot in the lab folder.

The detector may also flag an additional point if the local noise/window calculation identifies it. The required check is that all three deliberately injected latency spikes appear in the flagged sample list.

### 4. Review the detector — 8 minutes

Inspect this line:

```matlab
window = [14 0];
```

- `14` means fourteen preceding samples.
- `0` means no future samples.
- The current sample is included in the moving window.

Then inspect:

```matlab
p95Outlier = isoutlier(TT.P95Ms, 'movmedian', window);
```

This applies a local median baseline and scaled MAD threshold. It is more resistant to a single extreme point than a moving mean/standard-deviation baseline.

**Important operational distinction:** a centered window such as `[7 7]` can be useful for retrospective analysis, but it uses future samples. Do not use it as though it were a real-time detector.

### 5. Production interpretation — 4 minutes

Before turning an anomaly flag into an alert or automated action, consider:

- **Seasonality:** daily peaks and batch windows can make a single global baseline misleading.
- **Minimum sample count:** early windows have fewer observations.
- **Missing/duplicate samples:** validate time ordering and gaps before interpreting the detector.
- **Metric semantics:** p95 latency, CPU percentage and error percentage need different operational thresholds.
- **Alert fatigue:** require persistence, a rate-of-change condition or correlation with another signal where appropriate.
- **False positives:** a statistical outlier is a candidate for investigation, not proof of an incident.

Do not automatically restart a service or scale infrastructure because one point was flagged.

## Expected outcome

You should have:
- A deterministic 120-row synthetic telemetry CSV.
- A validated timetable with UTC sample times.
- Three deliberately injected p95 latency spikes detected.
- A separate CSV anomaly report.
- A plot suitable for a quick operator review.

## Quick self-check

1. Why use a timetable rather than an unlabelled numeric vector for timestamped operational metrics?
2. What does `[14 0]` mean, and why is it preferable to a centered window for online detection?
3. Why is median/MAD more robust than mean/standard deviation when a few extreme spikes are present?
4. Why should a flagged point not trigger automatic remediation by itself?
5. Which checks prevent a malformed CSV from producing misleading anomaly results?

## Production / interview relevance

This pattern is useful for analysing application latency, batch duration, infrastructure utilisation and transfer-volume telemetry. A strong engineer can explain the difference between a statistical anomaly and an actionable SLO breach, preserve the raw data, validate sample quality, and avoid look-ahead bias when designing online detection.

A concise interview answer:

**“I validate the time series first, use a trailing robust baseline to identify candidate anomalies, correlate them with service events and workload context, and only then decide whether an alert or controlled remediation is justified.”**

## Completion checkpoint

Report:
- MATLAB release and whether all required functions were available.
- CSV import and data-quality assertions: passed/failed.
- Whether samples 40, 85 and 100 were flagged.
- Number of total samples flagged.
- Whether the anomaly report and plot were created.
- One takeaway about trailing versus centred windows.

Do not paste production telemetry, private data or screenshots.

## 🧸 For Dummies

Imagine a service normally takes about three seconds to answer.

One request suddenly takes twenty seconds. That is unusual, so we mark it for investigation.

But we do not compare it with the average of the whole day if we are trying to detect problems as they happen. We compare it with the recent past.

The moving median is like asking: “What is normal for the last few minutes?”

If one measurement is wildly different, we flag it. We still check what happened before deciding that something is broken. 🧸
