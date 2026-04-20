# Predictive Process Monitoring for Penicillin Fermentation
 We use a real industrial-style penicillin fermentation dataset (time, OUR, CER, biomass, product concentration, oxygen efficiency, growth rate, etc.) to build a batch monitoring and yield forecasting system.
-Predicts biomass and penicillin concentration trajectory from any point mid-batch.
-Forecasts final yield early in the process, indicating batches likely to underperform.
## Bonus features:
-Meaningful engineered features (oxygen efficiency, cumulative feed, growth rate trends) that capture fermentation phase transitions without solving differential equations.
-Uncertainty quantification on all predictions, reflecting compounding simulation error over time.
-Can be extended into a physics-informed digital twin by coupling the delta model with a Monod kinetics layer.
## End product:
A reproducible pipeline that takes a partial batch trajectory as input and outputs: a simulated completion forecast with confidence bands, and a per-feature explanation of why this batch is trending the way it is.

