# Process Documentation

This document describes the steps taken in `Household_power_consumption.ipynb`: the dataset, the cleaning and transformation techniques, the EDA results and the key findings. Section numbers match the notebook.

## 1. Dataset overview

* **Source:** *Individual Household Electric Power Consumption*, UCI Machine Learning Repository (id 235), by Georges Hebrail and Alice Berard (EDF R&D). DOI [10.24432/C58K54](https://doi.org/10.24432/C58K54), licence CC BY 4.0.
* **Original data:** 2,075,259 one-minute readings from one house in Sceaux (7 km from Paris), December 2006 to November 2010.
* **Subset used:** 1 January to 30 June 2007, taken from the original text file and saved as `Dataset/power.csv`: 260,640 rows (181 days × 1,440 minutes) and 9 columns, plus an `index` column added by the exporter.
* **Variables:** `Date`, `Time`, `Global_active_power` (kW), `Global_reactive_power` (kVAR), `Voltage` (V), `Global_intensity` (A), `Sub_metering_1` (kitchen, Wh), `Sub_metering_2` (laundry room, Wh) and `Sub_metering_3` (electric water heater and air conditioner, Wh).
* **Data providers' note:** `Global_active_power × 1000 / 60 − Sub_metering_1 − Sub_metering_2 − Sub_metering_3` is the energy (Wh per minute) used by equipment that the three sub-meters do not cover. It is built as the feature `Unmetered_Wh`.

## 2. Data collection and exploration (notebook section 1)

**Loading.** The file is loaded with `pd.read_csv` exactly as delivered, with no type conversion, so that its original problems stay visible.

**Structure and data types (1.2).** The date, the time and six of the seven measurement columns are read as text, because the marker `?` stands for a missing reading. `Sub_metering_3` is already numeric because its missing cells are empty rather than `?`.

**Checks before changing anything (1.3):**

| Check | Result |
|---|---|
| Missing-value markers | `?` in six columns and empty cells in `Sub_metering_3` |
| Duplicates | No duplicated rows and no duplicated timestamps |
| Timeline | Every minute from 1 January to 30 June 2007 is present, with exactly one-minute steps |
| Skew | `Global_active_power` 1.70, `Global_intensity` 1.76, `Sub_metering_1` 5.36, `Sub_metering_2` 5.78 (right-skewed); `Voltage` -0.31 (nearly symmetric) |
| Zeros | Share of readings equal to 0: `Sub_metering_1` 91%, `Sub_metering_2` 68%, `Sub_metering_3` 66%, `Global_reactive_power` 25% |

## 3. Data cleaning and transformation (notebook section 2)

### 3.1 Parsing (2.1)

1. `Date` and `Time` are combined into one `DateTime` column with `pd.to_datetime` (format `dd-mm-yyyy hh:mm:ss`).
2. The seven measurement columns are converted to numbers with `pd.to_numeric(errors='coerce')`, so `?` becomes `NaN`.
3. `Date` and `Time` are dropped, `DateTime` becomes the index, and the data is sorted by time.

### 3.2 Missing values (2.2)

**Decision: drop every row with a missing value.** Imputation (filling the gaps) was considered and rejected.

| Finding | Value |
|---|---|
| Rows with missing data | 3,771 of 260,640 (**1.45%**) |
| Rows kept | **256,869 (98.55%)** |
| Rows with only some columns missing | **0**: all seven columns are missing together |
| Where the missing rows are | Three consecutive days have 864 to 1,440 missing rows each (29 April 1,440, 28 April 1,419, 30 April 864); every other day has 38 or fewer |
| The outage | One unbroken block from 28 April 00:21 to 30 April 14:23: 3,723 minutes (62 hours), **98.7% of all missing rows** |

**Why dropping, and not imputing:**

1. **Very little data is lost:** 1.45% of the rows.
2. **No partial rows:** a missing row has no usable values in any column, so nothing can be rescued from it.
3. **The missing data is mainly one long outage:** with no readings for 62 hours, any filled-in value would be invented.
4. **Imputation changes the statistics.** Compared with dropping, mean imputation leaves the means unchanged but shrinks every standard deviation by 0.73%. Forward fill and linear interpolation shift the column means by up to 1.45% and reduce the standard deviations by up to 0.70%. Forward fill also turns the outage into a flat line, and interpolation draws a straight line across it.
5. **Simplicity:** the analysis describes the data and finds patterns, so extra assumptions are not worth it.

**What dropping costs:** the timeline has gaps. 12 of the 181 days have fewer than 1,440 minutes, and 29 April has no data at all. Averages for those days use fewer readings.

### 3.3 Outlier detection and treatment (2.3)

* **Visual check:** box plots of all seven cleaned variables.
* **Statistical check:** the IQR rule. Values beyond 1.5 × IQR are *mild* outliers and beyond 3 × IQR are *extreme*.

| Column | Mild outliers | Mild % | Extreme outliers |
|---|---|---|---|
| `Global_active_power` | 13,664 | 5.32% | 2,075 |
| `Global_reactive_power` | 2,210 | 0.86% | 64 |
| `Voltage` | 570 | 0.22% | 0 |
| `Global_intensity` | 14,666 | 5.71% | 2,435 |
| `Sub_metering_1` | 23,199 | 9.03% | 23,199 |
| `Sub_metering_2` | 13,424 | 5.23% | 11,252 |
| `Sub_metering_3` | 0 | 0% | 0 |

**Plausibility checks.** A statistically unusual reading can still be a real event, so the extreme readings were tested against physics:

* 100% of the voltage readings are inside the usual European band of 207-253 V.
* Measured current matches power ÷ voltage with a correlation of 0.9992.
* In only 11 of 256,869 minutes does the sub-metered energy exceed the total energy, by at most 1.90 Wh (meter rounding).
* The highest active power is 10.67 kW (4 March 2007 at 19:34), a plausible appliance peak.

**Decision: the outliers are kept.** They are genuine consumption events, and removing them would erase the peak-demand behaviour the analysis is meant to study. For `Sub_metering_1` the IQR is zero because more than 90% of minutes read 0, so every non-zero reading counts as an "outlier"; the rule is not meaningful for such zero-inflated columns.

### 3.4 Feature engineering (2.4)

Five features are created from the `DateTime` index and the existing columns:

| Feature | How it is made | Why it is useful |
|---|---|---|
| `Hour` | Hour of the day (0-23) | Household routines follow the clock; the daily cycle is the strongest pattern |
| `TimeOfDay` | Night (0-5), Morning (6-11), Afternoon (12-17), Evening (18-23) | A readable categorical variable for charts and encoding |
| `IsWeekend` | 1 for Saturday and Sunday, else 0 | Weekend use differs from weekday use |
| `Month` | Month number (1-6) | Shows change across the six months |
| `Unmetered_Wh` | Active power in Wh per minute (kW × 1000 / 60) minus the three sub-meters, with small negative values clipped to 0 (meter rounding) | Energy used by appliances that are not on any sub-meter |

### 3.5 Encoding (2.5)

`TimeOfDay` is the only categorical variable, so it is the only one that needs encoding. **Label encoding** is used (Night = 0, Morning = 1, Afternoon = 2, Evening = 3). Its four groups have a natural order, and label encoding keeps that order in a single column. One-hot encoding is meant for unordered categories and would add three extra columns without adding information.

### 3.6 Normalisation and scaling

**Decision: scaling is not applied; the numerical features stay in their original units.**

1. Scaling matters only for methods that compare or combine columns numerically: distance-based methods (k-nearest neighbours, k-means), PCA, models trained by gradient descent, and regularised regression. This project uses none of them.
2. The methods used here are unaffected by scale. Correlation is scale-free: standardising the seven columns changes the correlation matrix by about 2 × 10⁻¹³. Histograms, box plots and the IQR rule work on one column at a time.
3. Original units keep the results interpretable. Findings such as "demand peaks at 2.20 kW at 21:00" or "54.5% of the energy is not metered" only make sense in kW, volts and watt-hours.
4. Scaling would distort the zero-inflated sub-meters. Standardising turns "appliance off" (0) into a negative number (about -0.2 for the kitchen).
5. `Unmetered_Wh` subtracts sub-meter energy (Wh) from active energy (kW converted to Wh). The subtraction is only valid in physical units.
6. If the data is used for clustering or modelling later, a scaler should be fitted on the training part at that point.

## 4. Exploratory data analysis (notebook section 3)

### 4.1 Descriptive statistics (3.1)

Statistics for the 256,869 cleaned rows (selected):

| Variable | Mean | Std | Median | Max |
|---|---|---|---|---|
| `Global_active_power` (kW) | 1.165 | 1.182 | 0.564 | 10.670 |
| `Global_reactive_power` (kVAR) | 0.124 | 0.112 | 0.104 | 1.148 |
| `Voltage` (V) | 239.209 | 3.593 | 239.610 | 250.890 |
| `Global_intensity` (A) | 4.975 | 4.999 | 2.600 | 46.400 |
| `Sub_metering_1` (Wh) | 1.332 | 6.705 | 0 | 78 |
| `Sub_metering_2` (Wh) | 1.671 | 6.631 | 0 | 78 |
| `Sub_metering_3` (Wh) | 5.832 | 8.187 | 0 | 20 |
| `Unmetered_Wh` | 10.581 | 11.437 | 5.567 | 122.167 |

Mean active power by category:

| Variable | Category | Rows | Mean active power (kW) |
|---|---|---|---|
| `TimeOfDay` | Night | 64,101 | 0.58 |
| | Morning | 64,078 | 1.28 |
| | Afternoon | 64,287 | 1.02 |
| | Evening | 64,403 | 1.78 |
| `IsWeekend` | 0 (weekday) | 186,329 | 1.05 |
| | 1 (weekend) | 70,540 | 1.47 |

### 4.2 Distributions (3.2)

Histograms with density curves and box plots by time of day and day of week:

* `Global_active_power` and `Global_intensity` are **right-skewed**: most minutes are low-demand and a smaller number are large bursts.
* `Voltage` is nearly symmetric and tightly clustered between 231 and 246 V.
* The sub-meters are **zero-inflated**. They behave like on/off appliance events rather than smooth quantities.
* Active power rises from night to morning, dips in the afternoon and peaks in the evening.

### 4.3 Relationships and correlation (3.3 and 3.4)

A scatter matrix of the seven measurements and a correlation heat map (the seven measurements plus `Unmetered_Wh`):

* `Global_active_power` and `Global_intensity` are essentially the same signal (r = 0.999), because current ≈ power ÷ voltage and voltage varies little.
* Among the sub-meters, `Sub_metering_3` correlates most strongly with total power (r = 0.61), then the kitchen (0.48) and the laundry room (0.47). No single appliance group explains total demand.
* `Voltage` is negatively related to load (r = -0.38): it sags slightly when the household draws more power.
* `Global_reactive_power` is only weakly related to active power (r = 0.28).
* `Unmetered_Wh` correlates with total power (r = 0.73) because it is the largest component of it.
* The three sub-meters are almost independent of each other (r between 0.07 and 0.13), so the appliance groups are used at different times.
* Correlation shows association, not causation.

### 4.4 Trends and patterns over time (3.5)

* An average daily load profile for weekdays and weekends.
* A heat map of mean active power by day of week and hour of day.
* A breakdown of the share of total energy by source and by hour of day.
* Mean daily energy on complete weekdays (124 days) and weekends (45 days).

## 5. Key findings (notebook section 3.6)

1. **A strong daily cycle.** Demand is lowest around 04:00 (0.49 kW) and highest at 21:00 (2.20 kW), a 4.5× swing. The three busiest hours are 19:00, 20:00 and 21:00, so the evening peak runs roughly from 19:00 to 21:00.
2. **Two peaks on weekdays.** A morning peak around 07:00 (1.70 kW) is followed by a lower midday plateau and the larger evening peak.
3. **Weekends differ.** The sharp weekday morning peak is absent, and demand is 89% higher between 11:00 and 16:00, which fits people being at home. A complete weekend day uses 34.4 kWh against 25.1 kWh on a weekday.
4. **The water heater and air conditioner group is the biggest metered load,** with 30.0% of all energy. It peaks at 09:00, the kitchen at 21:00 and the laundry room at 19:00, so the three groups follow different routines.
5. **More than half of the energy is not metered:** 54.5% comes from equipment outside the three sub-meters (for example most lighting, electronics and other appliances), so sub-metering alone cannot explain household demand.
6. **Voltage sags under load,** and current is almost a copy of power.

**What this means for decisions**

* **Peak shaving:** shifting flexible loads such as water heating or laundry away from 19:00-21:00 would flatten the evening peak.
* **Targeting savings:** the water heater and air conditioner circuit is the largest metered block, so it is the first place to look.
* **Monitoring gaps:** with 54% unmetered, sub-meters for lighting and electronics would give a fuller picture.

## 6. Why these variables were used

* **`Global_active_power`:** the main measure of consumption, so it is the centre of the analysis.
* **`Global_intensity`:** a near-duplicate of active power (r = 0.999). It confirms the physics but adds no separate information, so the two are treated as one signal.
* **`Sub_metering_1`, `2` and `3`:** different appliance groups, almost uncorrelated with each other, so each says something different about how the energy is used.
* **`Voltage`:** related to load but distinct (r = -0.38).
* **`Global_reactive_power`:** only weakly related to active power (r = 0.28), so it carries independent information.
* **`Hour`, `TimeOfDay`, `IsWeekend`, `Month`:** capture the daily, weekly and six-month patterns.
* **`Unmetered_Wh`:** the only feature that adds new information. It measures the 54.5% of the energy that the sub-meters do not cover.

## 7. Code documentation

* The code is commented step by step, and the comments explain what each step does and why.
* Markdown cells before and after the code explain the method and the reason for each choice.
* Captions are numbered automatically (`Table n.`, `Fig n.`) by a small helper function.
* The "Reading the distributions" and "Key insights" text is generated from the data with f-strings, so its numbers always match the analysis above it.


