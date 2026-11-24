# Lesson 012 — GNU Octave: detect telemetry anomalies with a causal moving-median baseline

**Date:** 2026-11-23 (updated 2026-11-24 for GNU Octave)  
**Domain:** Programming / GNU Octave / numerical analysis / observability  
**Duration:** 30 minutes  
**Environment:** GNU Octave Online (https://octave-online.net/) or GNU Octave 7.0+; base Octave only, no extra packages required. No MATLAB account, licence or local installation is needed.

**Compatibility note:** This lesson is written for GNU Octave. It deliberately avoids MATLAB-only features such as `timetable`, `table`, `readmatrix`, `writematrix` and `isoutlier`. It uses core Octave functions and a short, explicit rolling median/MAD detector so the logic is visible and portable.

## Objective

Build a small, reproducible telemetry-analysis job that generates synthetic CSV data, imports and validates it, detects latency/CPU/error-rate spikes using a causal moving-median and median-absolute-deviation (MAD) baseline, and exports a machine-readable anomaly report.

Practise:

1. Numeric CSV import/export with `csvread` and `csvwrite`.
2. Data-quality checks before analysis.
3. A trailing baseline that uses only earlier samples.
4. Robust outlier detection using median and MAD.
5. A separate report artifact and a plot for operator review.
6. Basic Octave scripting, loops, conditions, arrays and formatted output.

**Safety:** The dataset is generated locally and contains synthetic values. No production telemetry, endpoints, customer data or external services are used.

## Timed plan

| Time | Activity |
|---|---|
| 0–4 min | Open Octave Online and check the environment |
| 4–9 min | Create deterministic synthetic telemetry and CSV |
| 9–14 min | Import and validate the data |
| 14–22 min | Detect anomalies with a causal rolling baseline |
| 22–26 min | Export the anomaly report and plot the results |
| 26–30 min | Review operational caveats and self-check |

## Documentation

1. [GNU Octave — Getting Started](https://docs.octave.org/latest/Getting-Started.html) — using the Octave command line and scripts.
2. [GNU Octave — Statistics](https://docs.octave.org/latest/Statistics.html) — median and related statistical functions.
3. [GNU Octave — Two-Dimensional Plots](https://docs.octave.org/latest/Two_002dDimensional-Plots.html) — plotting and labelling results.

## Practical exercise

### 1. Preflight — 4 minutes

1. Open [Octave Online](https://octave-online.net/) in Firefox.
2. Wait for the Octave workspace to finish loading.
3. Click in the command window and run:

```octave
version
pwd
which median
which csvread
which csvwrite
which plot
```

Expected:
- Octave prints its version.
- `pwd` prints the current working directory.
- The four functions resolve to Octave-provided functions.
- No package installation or MathWorks account is required.

If the online service is unavailable, use the same commands in a locally installed GNU Octave environment. Do not install MATLAB or buy a licence for this lesson.

### 2. Create the lab script — 5 minutes

In Octave Online, create a new script in the editor and name it:

`telemetry_anomaly_lab.m`

Paste the complete script below into the editor and save it. If you prefer to start in the command window, you can paste the script there in sections, but saving it as a file is the intended programming exercise.

```octave
clear;
clc;
close all;

% Keep the lab deterministic so the expected anomalies are repeatable.
rand("seed", 42);

% Use the current working directory for all lab artifacts.
labDir = pwd();
csvPath = fullfile(labDir, "synthetic-telemetry.csv");
reportPath = fullfile(labDir, "anomaly-report.csv");

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

% Persist the generated data, then read the CSV back.
raw = [sample cpuPct p95Ms errorPct];
csvwrite(csvPath, raw);
data = csvread(csvPath);

% Data-quality gates before running the detector.
assert(rows(data) == n, "Unexpected row count in telemetry CSV.");
assert(columns(data) == 4, "Expected four numeric columns.");
assert(all(isfinite(data(:))), "CSV contains NaN or Inf values.");
assert(all(diff(data(:,1)) == 1), "Sample sequence has a gap or duplicate.");
assert(all(data(:,2) >= 0 & data(:,2) <= 100), ...
       "CPU percentage is outside 0..100.");
assert(all(data(:,3) >= 0), "Latency contains a negative value.");
assert(all(data(:,4) >= 0), "Error rate contains a negative value.");

% Extract columns from the imported CSV.
sample = data(:,1);
cpuPct = data(:,2);
p95Ms = data(:,3);
errorPct = data(:,4);

% Causal rolling median/MAD detector.
% For each current sample, use ONLY the previous 14 samples.
% The current sample is not allowed to influence its own baseline.
windowSize = 14;
madScale = 1.4826;
thresholdMultiplier = 3;

cpuOutlier = false(n,1);
p95Outlier = false(n,1);
errorOutlier = false(n,1);

for i = (windowSize + 1):n
    firstHistoryIndex = i - windowSize;
    lastHistoryIndex = i - 1;

    cpuHistory = cpuPct(firstHistoryIndex:lastHistoryIndex);
    p95History = p95Ms(firstHistoryIndex:lastHistoryIndex);
    errorHistory = errorPct(firstHistoryIndex:lastHistoryIndex);

    cpuMedian = median(cpuHistory);
    p95Median = median(p95History);
    errorMedian = median(errorHistory);

    cpuMAD = median(abs(cpuHistory - cpuMedian));
    p95MAD = median(abs(p95History - p95Median));
    errorMAD = median(abs(errorHistory - errorMedian));

    cpuThreshold = thresholdMultiplier * madScale * cpuMAD;
    p95Threshold = thresholdMultiplier * madScale * p95MAD;
    errorThreshold = thresholdMultiplier * madScale * errorMAD;

    cpuOutlier(i) = abs(cpuPct(i) - cpuMedian) > cpuThreshold;
    p95Outlier(i) = abs(p95Ms(i) - p95Median) > p95Threshold;
    errorOutlier(i) = abs(errorPct(i) - errorMedian) > errorThreshold;
end

anyAlert = cpuOutlier | p95Outlier | errorOutlier;

% Export a separate numeric report; preserve the source telemetry.
% Columns: sample, CPU flag, p95 flag, error flag, any-alert flag.
report = [sample cpuOutlier p95Outlier errorOutlier anyAlert];
csvwrite(reportPath, report);

alertSamples = sample(anyAlert);

fprintf("Samples analysed: %d\n", rows(data));
fprintf("Samples flagged:  %d\n", nnz(anyAlert));
fprintf("Flagged sample numbers: ");
fprintf("%d ", alertSamples);
fprintf("\nCSV:    %s\nReport: %s\n", csvPath, reportPath);

% Plot p95 latency and mark detected points.
figure(1);
plot(sample, p95Ms, "-");
hold on;
plot(sample(p95Outlier), p95Ms(p95Outlier), "ro", ...
     "markerfacecolor", "r");
grid on;
ylabel("p95 latency (ms)");
xlabel("Sample number (one-minute intervals)");
title("p95 latency: trailing moving-median/MAD outliers");
legend("p95 latency", "flagged samples", "location", "northeast");
hold off;
```

### 3. Run and validate the data — 5 minutes

In the Octave command window, run:

```octave
telemetry_anomaly_lab
```

If Octave says the script is not found, check the current directory with `pwd()` and make sure the editor saved `telemetry_anomaly_lab.m` in that working directory. You can also use the editor's Run control.

Expected:
- 120 samples analysed.
- The deliberately injected p95 latency spikes at samples 40, 85 and 100 should be flagged.
- Sample 85 should also show CPU and error-rate anomalies.
- The script creates `synthetic-telemetry.csv`, `anomaly-report.csv` and a plot in the current working directory.

The detector may flag additional points if the local noise/window calculation identifies them. The required check is that all three deliberately injected latency spikes appear in the flagged sample list.

### 4. Review the detector — 8 minutes

Inspect:

```octave
windowSize = 14;
```

The loop uses:

```octave
firstHistoryIndex = i - windowSize;
lastHistoryIndex = i - 1;
```

For the current sample `i`, this selects the fourteen observations immediately before it. The current observation is excluded from its own baseline, and no future observation is used. This is a causal detector suitable for an online-style check.

The detector calculates:

```octave
medianValue = median(history);
madValue = median(abs(history - medianValue));
threshold = 3 * 1.4826 * madValue;
```

- **Median:** the middle value in the recent history; less affected by extreme values than the mean.
- **MAD:** median absolute deviation; measures how spread out the recent values are around their median.
- **1.4826:** a conventional scale factor that makes MAD comparable to standard deviation for normally distributed data.
- **3:** the threshold multiplier. A point more than three scaled MADs from the baseline is flagged.

**Important operational distinction:** a centred window uses observations from both before and after the current sample. That can be useful for retrospective analysis, but it introduces look-ahead and must not be presented as a real-time detector.

### 5. Production interpretation — 4 minutes

Before turning an anomaly flag into an alert or automated action, consider:

- **Seasonality:** daily peaks and batch windows can make a short baseline misleading.
- **Warm-up period:** the first fourteen samples have no full preceding window and are not evaluated.
- **Missing/duplicate samples:** validate time ordering and gaps before interpreting the detector.
- **Metric semantics:** p95 latency, CPU percentage and error percentage need different operational thresholds.
- **Alert fatigue:** require persistence, a rate-of-change condition or correlation with another signal where appropriate.
- **False positives:** a statistical outlier is a candidate for investigation, not proof of an incident.
- **MAD equal to zero:** a perfectly flat history produces a zero threshold; production code should define a minimum threshold or fallback policy for that metric.

Do not automatically restart a service or scale infrastructure because one point was flagged.

## Expected outcome

You should have:
- A deterministic 120-row synthetic telemetry CSV.
- Data-quality checks that pass before analysis.
- A causal rolling median/MAD detector using only previous samples.
- The three deliberately injected p95 latency spikes detected.
- A separate numeric CSV anomaly report.
- A plot suitable for a quick operator review.

## Quick self-check

1. Why validate the CSV before running the detector?
2. Why does the rolling baseline use samples `i-14` through `i-1`, rather than including the current sample?
3. Why is median/MAD more robust than mean/standard deviation when a few extreme spikes are present?
4. What does the factor `1.4826` do?
5. Why should a flagged point not trigger automatic remediation by itself?
6. What is the trade-off of using a fourteen-sample history?

## Production / interview relevance

This pattern is useful for analysing application latency, batch duration, infrastructure utilisation and transfer-volume telemetry. A strong engineer can explain the difference between a statistical anomaly and an actionable SLO breach, preserve the raw data, validate sample quality, and avoid look-ahead bias when designing online detection.

A concise interview answer:

**“I validate the time series first, use a trailing robust baseline to identify candidate anomalies, correlate them with service events and workload context, and only then decide whether an alert or controlled remediation is justified.”**

## Completion checkpoint

Report:
- Octave version and whether the required functions resolved.
- CSV import and data-quality assertions: passed/failed.
- Whether samples 40, 85 and 100 were flagged.
- Number of total samples flagged.
- Whether the anomaly report and plot were created.
- One takeaway about excluding the current/future samples from a real-time baseline.

Do not paste production telemetry, private data or screenshots.

## 🧸 For Dummies

Imagine a service normally takes about three seconds to answer.

One request suddenly takes twenty seconds. That is unusual, so we mark it for investigation.

But we do not compare it with the average of the whole day if we are trying to detect problems as they happen. We compare it with the recent past.

The moving median is like asking: “What has been normal during the last fourteen measurements?”

We deliberately leave the current measurement out of the comparison. Otherwise, the unusually slow request could influence the definition of normal.

If one measurement is wildly different, we flag it. We still check what happened before deciding that something is broken. 🧸
