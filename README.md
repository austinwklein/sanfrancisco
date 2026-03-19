# SFO Fog & Delay Predictor

**CSCE A462 — Data Mining | Assignment 1 | University of Alaska Anchorage**

###### **Disclaimer: AI was used for assistance in writing this readme file**

## Aim

San Francisco International Airport (SFO) is one of the busiest airports in the United States and is notorious for weather-related operational disruptions — particularly the marine layer fog known locally as "Karl the Fog." This project asks: **can hourly weather conditions at SFO predict departure delays?**

This question is relevant to anyone flying out of SFO, anyone living near the airport, and anyone studying the relationship between microclimatic conditions and aviation system performance. The analysis focuses on United Airlines departures in 2023, which represent ~54% of all SFO operations, making them a strong proxy for airport-wide delay behavior.

### Deployment Architecture

```
NOAA LCD (hourly wx) ──────┐
                           ├──► KNIME Workflow ──► Linear Regression ──► Delay Prediction
BTS Departures (per-flight)┤                  └──► k-Means Clustering ──► Delay Profiles
                           │
SFO Landing Stats (monthly)┘
```

---

## Data Sources

| Dataset | Source | Granularity | Role |
|---|---|---|---|
| NOAA Local Climatological Data (KSFO) | [NOAA LCD](https://www.ncdc.noaa.gov/cdo-web/datasets/LCD/stations/WBAN:23234/detail) | Hourly | Weather features (visibility, humidity, wind, fog) |
| BTS Detailed Statistics Departures | [Bureau of Transportation Statistics](https://www.transtats.bts.gov/ontime/Departures.aspx) | Per-flight | Departure delay ground truth |
| Air Traffic Landings Statistics | [DataSF](https://data.sfgov.org/Transportation/Air-Traffic-Landings-Statistics/fpux-q53t) | Monthly | Traffic volume feature |

### Data Notes

- NOAA LCD covers 2015–present but this analysis uses **2023** to match BTS data availability
- BTS data is filtered to **United Airlines (UA)** departing from **SFO**
- SFO Landing Statistics are filtered to **2023** and aggregated to monthly totals
- The raw NOAA file contains mixed record types (`FM-15` hourly METARs, `FM-12` synoptic, `SOD` daily summaries). Only `FM-15` records are used.

---

## Setup & Reproduction

### Prerequisites

- [KNIME Analytics Platform](https://www.knime.com/downloads) (v5.x)
- KNIME Python Integration extension (install via Help → Install New Software)
- Python 3.x with `pandas` installed (This should be bundled with knime and prompt you to set up when opening this project)

### Data Download 
###### (Also all bundled in repo under /data)

1. **NOAA LCD**: Go to https://www.ncdc.noaa.gov/cdo-web/datatools/lcd, select station `KSFO - San Francisco International Airport`, date range 2023, download CSV. Save as `data/SFO_Climate_NOAA.csv`.

2. **BTS Departures**: Go to https://www.transtats.bts.gov/OT_Delay/OT_DelayCause1.asp, select Origin Airport = SFO, Airline = United Airlines, Year = 2023, download Detailed Statistics. Save as `data/Detailed_Statistics_Departures.csv`.

3. **SFO Landing Stats**: Go to https://data.sfgov.org/Transportation/Air-Traffic-Landings-Statistics/fpux-q53t, export as CSV. Save as `data/Air_Traffic_Landings_Statistics.csv`.

### Running the Workflow

1. Open KNIME and import the workflow from `knime_workspace/SFO_A1/`
2. Update file paths in the three Python Script nodes to match your local `data/` directory
3. Execute all nodes

---

## KNIME Workflow

```
[NOAA Python Script] ──────────────────────────────────┐
                                                        ▼
[BTS Python Script] ──────────────────────────────► [Joiner] ──► [Date&Time Part Extractor]
                                                                           │
[Air Traffic Landings Python Script] ──────────────────────────────────────┤
                                                                           ▼
                                                                     [Joiner #2]
                                                                           │
                                                          ┌────────────────┤
                                                          ▼                ▼
                                              [Linear Regression    [Missing Value]
                                                  Learner]               │
                                                     │                    ▼
                                                     ▼               [k-Means]
                                             [Regression                  │
                                              Predictor]                  ▼
                                                                   [Color Manager]
                                                                          │
                                                                          ▼
                                                                   [Scatter Plot]
```

### Key Node Configurations

**Python Script nodes** use `pandas` with `dtype=str` to bypass KNIME's CSV type inference issues with the raw NOAA and BTS files.

**Joiner #1** joins NOAA hourly weather with BTS hourly delay buckets on `date_only + hour`.

**Joiner #2** joins the result with monthly landing counts on `month`.

**Linear Regression Learner** target: `avg_delay`. Features: `HourlyVisibility`, `HourlyRelativeHumidity`, `HourlyWindSpeed`, `is_fog`, `monthly_landings`, `month`.

**k-Means** uses k=3 clusters on the same feature set plus `avg_delay`.

---

## Preprocessing Steps

All preprocessing is performed inside KNIME Python Script nodes:

**NOAA data:**
- Filter to `REPORT_TYPE == FM-15` (hourly METAR observations only)
- Filter to year 2023
- Parse `DATE` to extract `date_only` and `hour`
- Convert weather columns to numeric
- Flag fog hours: `is_fog = 1` where `HourlyPresentWeatherType` contains `FG`

**BTS data:**
- Skip 7 metadata header rows
- Parse `Date (MM/DD/YYYY)` to extract `date_only` and `hour` from scheduled departure time
- Aggregate to hourly buckets: `avg_delay`, `total_flights`, `pct_delayed`

**Landing stats:**
- Filter to 2023
- Aggregate to monthly totals: `monthly_landings`

---

## Findings

### Regression Results

| Variable | Coefficient | p-value | Interpretation |
|---|---|---|---|
| HourlyVisibility | -2.586 | < 0.001 | Each 1 mile drop in visibility adds ~2.6 min delay |
| HourlyWindSpeed | +0.480 | < 0.001 | Each 1 kt of wind adds ~0.5 min delay |
| monthly_landings | +0.002 | < 0.001 | Higher traffic months increase delay cascading |
| month | -0.516 | < 0.001 | Seasonal confounding — later months slightly reduce predicted delay |
| is_fog | -21.994 | 0.001 | True IFR fog events reduce average delay (flights cancelled, not delayed) |
| HourlyRelativeHumidity | +0.018 | 0.433 | Not statistically significant |

### Notable Findings

1. **Visibility is the dominant weather predictor** — a 4-mile visibility reduction (10SM to 6SM) predicts ~10 additional minutes of average departure delay per hour.

2. **The fog paradox** — hours where actual IFR fog is coded (`is_fog = 1`) show *lower* average delay, not higher. This is because the BTS dataset records cancellations separately from delays. When conditions are severe enough to trigger a Ground Delay Program, flights are cancelled rather than delayed — and cancelled flights don't contribute to average delay minutes. This is a meaningful operational insight, not a data artifact.

3. **Traffic volume matters independently of weather** — the monthly landing count coefficient is small but highly significant. Summer months see both higher traffic and better weather; the model separates these effects.

4. **Humidity alone is not predictive** — it is the visibility effect of fog, not atmospheric moisture per se, that drives delays.

### Clustering Results

k-Means with k=3 produced three operationally meaningful clusters:

| Cluster | Avg Visibility | Avg Delay | Interpretation |
|---|---|---|---|
| cluster_0 | 9.16 SM | 76.4 min | High-delay hours — disrupted operations |
| cluster_1 | 9.61 SM | 3.7 min | Normal operations — majority of hours |
| cluster_2 | 9.99 SM | 9.4 min | Best-case operations — clear and light traffic |

The scatter plot of `HourlyVisibility` vs `avg_delay` colored by cluster shows that while low-visibility hours are predominantly in the high-delay cluster, high-delay hours also occur at 10SM visibility — confirming that wind, traffic cascading, and other factors contribute independently of fog.

---

## Limitations & Future Work

- **Airline scope**: Analysis is limited to United Airlines departures. A full picture would include all carriers.
- **Cancellations**: BTS delay data does not include cancelled flights as delay events, which distorts the `is_fog` coefficient.
- **Granularity**: Weather observations are hourly; actual block-time delays are point-in-time. A finer-grained join using actual departure timestamps would improve model accuracy.
- **Model complexity**: Linear regression assumes linear relationships. A Random Forest or Gradient Boosting model would likely capture non-linear threshold effects (e.g., visibility below 1 mile triggering ILS approaches and dramatically reduced throughput).
- **KNIME Python Integration**: The bundled Python environment requires manual output port configuration. A future iteration could use pre-cleaned CSVs fed directly into KNIME CSV Reader nodes for simpler reproducibility.

---

## Acknowledgments

- **Weather data:** _NOAA National Centers for Environmental Information_
- **Flight operations data:** _Bureau of Transportation Statistics_
- **Landing statistics:** _The City and County of San Francisco via DataSF (data.sfgov.org)_