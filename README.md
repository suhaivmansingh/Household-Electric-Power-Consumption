# Household Electric Power Consumption: Data Analysis and Visualization

Data cleaning, transformation, exploratory data analysis (EDA) and visualisation of one household's electricity use, based on one-minute readings from January to June 2007. The whole analysis is in one notebook, [`Household_power_consumption.ipynb`](Household_power_consumption.ipynb).

The project covers loading and exploring the data, cleaning it, creating features, exploring patterns with statistics and plots, and drawing insights.

## Objective

Understand how the household uses electricity over time and what drives that use: daily and weekly routines, peak demand, and how total consumption relates to the individual appliance groups.

## Dataset

| | |
|---|---|
| **Name** | Individual Household Electric Power Consumption |
| **Creators** | Georges Hebrail (EDF R&D, Clamart, France) and Alice Berard (TELECOM ParisTech) |
| **Repository** | UCI Machine Learning Repository, dataset id 235 |
| **Original data** | 2,075,259 one-minute measurements from a house in Sceaux (7 km from Paris), December 2006 to November 2010 |
| **Used here** | 1 January to 30 June 2007: **260,640 rows** (181 days × 1,440 minutes), in `Dataset/power.csv` |

**Citation:** Hebrail, G. & Berard, A. (2006). *Individual Household Electric Power Consumption* [Dataset]. UCI Machine Learning Repository. https://doi.org/10.24432/C58K54

| Variable | Unit | Meaning |
|---|---|---|
| `Date`, `Time` | dd-mm-yyyy, hh:mm:ss | Timestamp of the one-minute reading |
| `Global_active_power` | kW | Minute-averaged active power (energy actually consumed) |
| `Global_reactive_power` | kVAR | Minute-averaged reactive power |
| `Voltage` | V | Minute-averaged voltage |
| `Global_intensity` | A | Minute-averaged current |
| `Sub_metering_1` | Wh | Kitchen (dishwasher, oven, microwave) |
| `Sub_metering_2` | Wh | Laundry room (washing machine, tumble drier, refrigerator, light) |
| `Sub_metering_3` | Wh | Electric water heater and air conditioner |

## What the notebook does

| Notebook section | Task requirement | What is done |
|---|---|---|
| 0. Dataset source and background | Dataset overview | Source, licence, variable dictionary |
| 1.1 Dataset loading | Load with Pandas | `pd.read_csv` |
| 1.2 Initial exploration | Structure: columns, rows, data types | Shape, data types, non-null counts, distinct values |
| 1.3 Summary statistics and problems | Summary statistics, missing values, outliers, inconsistent data | `?` markers, duplicates, timeline completeness, skew, share of zeros |
| 2.1 Parsing | Data types | One `DateTime` index; text columns converted to numbers |
| 2.2 Missing values | Handle missing values | Rows with missing values dropped, with the reasoning |
| 2.3 Outliers | Detect and handle outliers | Box plots, IQR rule, plausibility checks; outliers kept |
| 2.4 Feature engineering | Create new features | `Hour`, `TimeOfDay`, `IsWeekend`, `Month`, `Unmetered_Wh` |
| 2.5 Encoding | Encode categorical variables | Label encoding of `TimeOfDay` |
| 3.1 Descriptive statistics | Summary statistics for numerical and categorical variables | `describe()` and category frequency tables |
| 3.2 Distributions | Histograms, density plots | Histograms with density curves, box plots by time and day |
| 3.3 Relationships | Scatter plots, pair plots | Scatter matrix |
| 3.4 Correlation | Heatmaps, correlation matrix | Correlation heatmap |
| 3.5 Trends and patterns | Patterns and trends | Weekday/weekend load profile, hour-by-day heat map, energy breakdown |
| 3.6 Key insights | Key insights | Findings and decision-oriented recommendations |

## Key results

* **Missing data:** 3,771 of 260,640 rows (1.45%) have no reading. **98.7% of them are one unbroken 62-hour outage** (28 April 00:21 to 30 April 14:23). All seven columns are always missing together. The rows are dropped, leaving **256,869 rows**.
* **Daily cycle:** demand is lowest around 04:00 (0.49 kW) and highest at 21:00 (2.20 kW), a 4.5× swing. The evening peak runs from about 19:00 to 21:00, and weekdays add a morning peak around 07:00.
* **Weekends:** a complete weekend day uses 34.4 kWh against 25.1 kWh on a weekday. Demand between 11:00 and 16:00 is 89% higher at weekends.
* **Where the energy goes:** the water heater and air conditioner group is the largest metered load (30.0% of all energy). **54.5% of the energy is not measured by any sub-meter.**
* **Physics check:** `Global_active_power` and `Global_intensity` are essentially the same signal (r = 0.999), and voltage sags as load rises (r = -0.38).
* **Outliers:** the high readings are genuine appliance peaks (all voltages are within 207-253 V, and measured current matches power ÷ voltage with r = 0.9992), so they are kept.
* **Sub-meters:** they are zero-inflated: 91% of minutes read 0 for the kitchen, 68% for the laundry room and 66% for the water heater and air conditioner.

## Repository structure

```
.
├── Household_power_consumption.ipynb   # the full analysis, with outputs
├── README.md
├── Process_Documentation.md            #Document to explain contents of notebook
└── Dataset/
    ├── power.csv                       # original six-month subset (unchanged)
    └── power_clean.csv                 # cleaned dataset saved by the notebook
```

## Setup and run

Requires Python 3.10 or newer (the notebook was developed with Python 3.13 in Anaconda). The notebook uses these libraries: `pandas`, `numpy`, `matplotlib` and `seaborn`. There are two ways to run it. Choose one.

**First, get the code (both options):**

```bash
git clone https://github.com/<your-username>/<repository-name>.git
cd <repository-name>
```

### Option A: Anaconda or an existing Python installation (no virtual environment)

This is how the notebook was developed and run. It uses the `base` kernel (Python 3.13, Anaconda), which already includes the required libraries.

1. Open `Household_power_consumption.ipynb` in Jupyter, JupyterLab or VS Code.
2. Select the **base** Python kernel. In VS Code this is *Select Kernel* in the top right of the notebook, then the `base (Python 3.13) ~\anaconda3\python.exe` entry.
3. If a library is missing, install it into that environment:

   ```bash
   pip install pandas numpy matplotlib seaborn
   ```

4. Choose **Run > Run All Cells**.

### Option B: A virtual environment (keeps the project's libraries separate)

```bash
# Create and activate a virtual environment
python -m venv .venv
source .venv/bin/activate          # Windows PowerShell: .venv\Scripts\Activate.ps1

# Install the libraries the notebook uses
pip install pandas numpy matplotlib seaborn jupyterlab ipykernel

# Open the notebook and choose Run > Run All Cells
jupyter lab Household_power_consumption.ipynb
```

In VS Code, select the `.venv` interpreter as the notebook kernel instead of `base`.

### Notes for both options

* The notebook reads `Dataset/power.csv`, so open it from the repository root (the folder that contains `Dataset/`).
* Running all cells takes under a minute.


## Design decisions

* **Rows with missing values are dropped, not imputed.** Only 1.45% of rows are affected, nearly all of them one 62-hour outage, so any filled-in values would be invented. Imputation would also change the statistics.
* **Outliers are kept.** They are real consumption events and the analysis is about peak demand.
* **Label encoding for `TimeOfDay`.** Its four groups have a natural order (Night, Morning, Afternoon, Evening), and it needs only one column.
* **No scaling.** No distance-based method or model is used, correlation is scale-free, and original units keep the results interpretable. See the process documentation for the full reasoning.

## Limitations

* One household, six months, one climate: the findings describe this household and should not be generalised without more data.
* The missing rows were dropped, not repaired, so the timeline has gaps (29 April has no data).
* The relationships found are correlations, not proof of cause.

