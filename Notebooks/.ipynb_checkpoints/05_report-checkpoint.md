# Predictive Process Monitoring for Penicillin Fermentation
## ML Component — Technical Summary

---

## 1. Model Summary

Two Random Forest **delta models** were trained to predict the one-step-ahead
change in penicillin concentration and biomass respectively. Predicting the
incremental change (Δ) rather than absolute concentration is more stable given
the non-stationary nature of batch fermentation trajectories.

At inference time, the models run in a **simulation loop** from a chosen
observation cutoff. All sensor features (OUR, CER, DO, temp, pH, etc.) are
carried forward from real observed data via **teacher forcing** — only
`penicillin_lag1` and `biomass_lag1` are updated from model predictions,
reflecting the real monitoring scenario where sensors stream continuously but
penicillin concentration is unknown post-cutoff.

**Monte Carlo uncertainty** is applied by sampling individual decision trees
at each simulation step rather than using the ensemble mean. Fifty trajectories
yield a 90% CI that widens naturally over the forecast horizon.

---

## 2. Performance

| Metric | Penicillin | Biomass |
|---|---|---|
| Test RMSE (80/20) | 0.0065 g/L | 0.0023 g/L |
| LOBO-CV Mean RMSE | 0.0075 g/L ± 0.0028 | 0.0020 g/L ± 0.0007 |

**Leave-One-Batch-Out Cross Validation (LOBO-CV)** was used as the primary
generalisation metric. Each of the ten batches is held out in turn and the
model retrained on the remaining nine. The LOBO-CV RMSE closely matches the
80/20 test RMSE, indicating the model is not severely overfitting.

---

## 3. Forecast Output

The forecast plot shows:
- **Solid blue line** — ensemble mean forecast (average of 300 trees)
- **Shaded band** — 90% CI from 50 Monte Carlo trajectories
- **Dashed coral line** — actual measured values (validation only)
- **Vertical dotted line** — observation cutoff t_observe

Numerical outputs reported:
- Forecasted **peak** penicillin with 90% CI and timestamp
- Forecasted **end-of-batch** penicillin with 90% CI

**Observation cutoff set to t = 40h** — approximately 16–20% into the batch.
This is deliberately early so the forecast is actionable: operators receive
a yield trajectory while the majority of the batch remains ahead of them.

---

## 4. Feature Importance — Biological Interpretation

| Feature | Importance | Biological Rationale |
|---|---|---|
| `oxygen_efficiency` | Highest | Captures OUR/aeration balance — high efficiency signals active penicillin synthesis phase |
| `growth_rate` | High | Penicillin is a secondary metabolite; high growth rate competes for precursors, suppressing yield |
| `time` | High | Encodes fermentation phase implicitly — accumulation vs. degradation phase |
| `Base flow rate (Fb)` | Moderate | Controls pH via acid neutralisation; pH directly affects penicillin synthesis enzyme activity |
| `our` / `our_lag1` | Moderate | Direct proxy for metabolic activity; lag captures rate of change |
| `penicillin_lag1` | Moderate | Local baseline for delta prediction; primary source of compounding error in simulation |

### Features Excluded

- **Substrate concentration** — showed dominant importance (0.49) but was
  removed as a likely leakage concern: it encodes full batch trajectory
  information in the Pensim simulation framework.
- **PAA (phenylacetic acid)** — the direct biosynthetic precursor to penicillin
  and biologically the most important excluded variable. Omitted because it is
  an offline measurement in this dataset, incompatible with real-time monitoring.
  Should be the first feature added if inline PAA sensing becomes available.

---

## 5. Modelling Decisions

### Delta formulation
Absolute concentration prediction → poor simulation stability (learns
batch-specific levels). Log-ratio → numerical instability near zero.
**Delta formulation** → best RMSE, simulation stability, and interpretability.

### Random Forest configuration

| Parameter | Value | Rationale |
|---|---|---|
| `n_estimators` | 300 | Larger pool for stable MC uncertainty bands |
| `max_depth` | 12 | Prevents memorisation of batch-specific trajectories |
| `min_samples_leaf` | 3 | Smooths delta predictions, reduces step discontinuities |
| `max_features` | 0.6 | Decorrelates trees; critical with correlated features like OUR/oxygen_efficiency |

These values are a validated starting configuration. Further tuning via grid
search over `max_depth` (8–16) and `min_samples_leaf` (1–5) could improve
penicillin model variance across folds.

---

## 6. Limitations

**Small batch cohort (n = 10).** Training on 8 batches means the test set
(batches 9–10) may not represent the full range of batch behaviours. In
practice this produced a consistent upward bias in penicillin forecasts for
the test batches — the model extrapolates dynamics from higher-yield training
batches. LOBO-CV partially mitigates this but does not resolve the fundamental
data scarcity.

**Penicillin degradation phase.** The model captures accumulation well but
underestimates the late-phase decline. Penicillinase-driven degradation does
not produce a strong signal in available sensor features — OUR and CER decline
ambiguously in the late phase. This is a missing variable problem. Future work
could incorporate pH excursions or viscosity as late-phase indicators.

---

## 7. Real-World Application

Deployed as a **soft sensor and monitoring dashboard**, the system would:

1. Ingest real-time sensor readings at each 0.2h interval
2. Output a trajectory forecast to batch end with 90% CI
3. Flag early yield direction at t = 40h
4. Provide a SHAP attribution explaining which sensor readings are driving
   the forecast — giving operators a biological explanation, not a black box

The system is a **monitoring and forecasting tool**, not an optimiser. Control
variables (RPM, aeration, feed rate) showed near-zero variance across batches
in this dataset, making sensor-driven dynamics the primary predictive signal.
An optimisation layer could be built on top once a dataset with deliberate
setpoint variation is available.

The architecture is extensible toward a **physics-informed digital twin** by
coupling the delta model with a Monod kinetics constraint layer, which would
reduce compounding simulation error and provide a more principled uncertainty
basis.
