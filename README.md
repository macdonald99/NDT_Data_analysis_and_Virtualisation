# Data Analysis and Visualisation
## South Africa: Financial Inclusion & Agricultural R&D Investment

**Module:** Data Analysis And Visualization
**Programme:** Diploma in ICT
**Examiner:** Melvin Kisten
**Student:** Macdonald Mako — 202441615

---

## Project Overview

For this project we analysed two World Bank datasets, focusing on South Africa, to understand the connection between financial inclusion for business/farm financing and investment in agricultural research and innovation. The datasets being:

1. **Global Findex** — "Borrowed to start, operate, or expand a farm or business (% age 15+)"
   Source: Global Financial Inclusion (Findex) Database, World Bank

2. **ASTI (Agrifood Systems Technologies and Innovations)** — Agricultural R&D spending and researcher indicators
   Source: International Food Policy Research Institute (IFPRI)

**Research question:** Does improved access to business/farm financing in South Africa align with investment in agricultural research and innovation over time?

---

## Repository Structure

```
NDT_Data_analysis_and_Virtualisation/
├── data/
│   ├── raw/                  # Original downloaded CSVs (unedited)
│   │   ├── WB_GS_FIN21_T_A.csv
│   │   └── IFPRI_ASTI.csv
│   └── processed/            # Cleaned/exported data
│       ├── findex_clean.csv
│       ├── asti_clean.csv
│       └── asti_analysis.xlsx
├── notebooks/
│   └── analysis.ipynb        # Full analysis pipeline
├── db/
│   └── project.db            # SQLite database
├── requirements.txt
└── README.md
```

---

## Environment Configuration

**Requirements:**
- Python 3.10+ (developed and tested on Python 3.14)
- pip

**Setup instructions:**

1. Clone the GitHub repository:
   ```
   git clone https://github.com/macdonald99/NDT_Data_analysis_and_Virtualisation.git
   cd NDT_Data_analysis_and_Virtualisation
   ```

2. Install all necessary dependencies:
   ```
   pip install -r requirements.txt
   pip install openpyxl
   ```

3. Launch Jupyter:
   ```
   jupyter notebook
   ```

4. Open `notebooks/analysis.ipynb` and run all cells in order (Kernel → Restart & Run All).

We then opened the notebook, `notebooks/analysis.ipynb` to be specific, and ran all the cells in order — from Kernel → Restart & Run All.

---

## Key Findings

- The gender gap is rather tenacious in business/farm financing access, where males borrowed at roughly **1.6x** the rate of females in both 2014 (9.25% vs 5.94%) and 2017 (7.19% vs 4.22%).
- Although volatile year after year and not steady, Agricultural R&D spending (% of agricultural GDP) has more than doubled from 1981 (1.22%) to its 2013 peak (3.06%).
- Over the same period, the number of full-time equivalent agricultural researchers increased relatively little, with a poor connection (**r = 0.24**) to spending, indicating that investment increased more quickly than the research workforce.
- There is only one overlapping year (2014) between the two datasets, which restricts a direct year-by-year association between R&D spending and financial inclusion; this constraint is covered below.

---

## Data Limitations

- ASTI data is available annually from 1981 to 2014, however Global Findex data for South Africa is only available for survey years 2014 and 2017 (2 data points). A statistically sound correlation between the two datasets could not be established because of this mismatch.
- According to the source methodology, all monetary ASTI figures are in PPP-adjusted 2021 constant dollars.

---

## Error Handling

The notebook includes the following safeguards:

- A SELECT query with the same WHERE clause usually comes before database UPDATE/DELETE operations, in order to verify scope before changing data.
- It explicitly checks for missing values (`.isnull().sum()`) prior to making any cleaning decisions, as opposed to making assumptions.
- To avoid quietly merging incompatible measures (e.g. currency values with percentages), pivot procedures include all pertinent grouping keys (e.g. `unit` alongside `indicator`).

---

## Methodology — What We Did, How We Did It, and What We Found

A step-by-step walkthrough of the full pipeline in `notebooks/analysis.ipynb`, explaining every cell, why it was used, and the result it produced.

### 1. Setup

```python
import pandas as pd
import numpy as np
import matplotlib.pyplot as plt
import seaborn as sns
```

We used four libraries, which are the core of the pipeline. `pandas` (aliased `pd`) handles tables of data (DataFrames) and was mainly used for loading, cleaning, and manipulating data. `numpy` (aliased `np`) handles fast mathematical operations on arrays. `matplotlib.pyplot` (aliased `plt`) is the plotting library used for every chart. `seaborn` (aliased `sns`) works with matplotlib for nicer statistical styling.

**Result:** All four loaded without error, confirmed with a simple print statement before anything else was built on top of them.

### 2. Loading the raw datasets

```python
findex_raw = pd.read_csv('../data/raw/WB_GS_FIN21_T_A.csv')
asti_raw = pd.read_csv('../data/raw/IFPRI_ASTI.csv')
```

We first read the CSV file off disk into a DataFrame — basically a table object that can be filtered, sorted, and analysed in Python. The notebook is portable for anyone who clones the GitHub repository, since the `../` in the path goes back up from `notebooks/` to the project root before entering `data/raw/`.

**Result:** `Findex shape: (50880, 35)` and `ASTI shape: (8431, 38)`. This confirmed both files loaded correctly, and revealed that both cover every country in the world, not just South Africa — setting up the next step.

### 3. Filtering to South Africa

```python
findex_zaf = findex_raw[findex_raw['REF_AREA'] == 'ZAF'].copy()
asti_zaf = asti_raw[asti_raw['REF_AREA'] == 'ZAF'].copy()
```

By means of boolean filtering — `findex_raw['REF_AREA'] == 'ZAF'` — the code produces a True/False value for every row, True where the country code is South Africa, and pandas keeps only the True rows. `.copy()` ensures the result is a fully independent table rather than a view referencing the original — without it, later edits could trigger warnings or unintended side effects on the original data.

**Result:** `Findex (South Africa only): (192, 35)` and `ASTI (South Africa only): (136, 38)` — the exact subset of rows belonging to South Africa, out of tens of thousands covering every country.

### 4. Inspecting the raw data before cleaning

```python
findex_zaf[['REF_AREA_LABEL', 'SEX_LABEL', 'TIME_PERIOD', 'OBS_VALUE']].head(20)
```

The use of `.head(20)` shows the first 20 rows without printing all 192; selecting just four columns eliminates the metadata noise so that the actual content — country, sex breakdown, year, value — is visible and usable, this is also done for accuracy's sake.

**Result:** Page after page of `NaN` (pandas' missing-value marker) for `OBS_VALUE`, from 1960 through 1966 and beyond. This is how we accurately discovered, directly, that Findex survey data isn't collected every year — the finding that justified the cleaning decision below.

### 5. Selecting and renaming columns

```python
findex_clean = findex_zaf[['SEX_LABEL', 'TIME_PERIOD', 'OBS_VALUE']].copy()
findex_clean.columns = ['sex', 'year', 'pct_borrowed']

asti_clean = asti_zaf[['INDICATOR_LABEL', 'UNIT_MEASURE_LABEL', 'TIME_PERIOD', 'OBS_VALUE']].copy()
asti_clean.columns = ['indicator', 'unit', 'year', 'value']
```

This drops every column except the 3-4 that are of significance, and renames them from World Bank's shouting-case codes (`OBS_VALUE`, `TIME_PERIOD`) to short, meaningful, lowercase names, making every subsequent line of code self-explanatory without needing to memorize World Bank's internal column codes.

### 6. Checking and handling missing values

```python
findex_clean.isnull().sum()
asti_clean.isnull().sum()
```

`.isnull()` flags every cell True/False, True = missing, while `.sum()` counts the Trues per column.

**Result:** Findex's `pct_borrowed` had **186 missing out of 192 rows** — numerically confirming what the `.head(20)` inspection had already shown visually. ASTI has **zero missing values** anywhere.

```python
findex_clean = findex_clean.dropna(subset=['pct_borrowed'])
```

`.dropna(subset=[...])` eliminates just the rows that don't have that particular value. Since there is no reliable means to estimate a survey response for a year when the survey was not conducted, we chose to drop rather than guess-fill.

**Result:** Findex dropped from 192 rows to exactly **6 rows** — 2014 and 2017, each split into Total/Female/Male.

### 7. Descriptive statistics

```python
findex_clean.groupby('sex')['pct_borrowed'].describe()
asti_clean.groupby(['indicator', 'unit'])['value'].describe()
```

`.groupby()` divides data into relevant buckets before summarizing, so instead of combining unrelated measures, `.describe()` (count, mean, std, min, max, quartiles) is computed within each group.

**Result:**
- **Findex:** In both survey years, there was a consistent gender disparity of almost 3 percentage points (men borrowed at about 1.6 times the female rate), with the male average being 8.22% and the female average being 5.08%.
- **ASTI:** FTE researcher headcount ranged from around 640 to about 960, a significantly smaller proportional increase, whereas R&D spending as a percentage of agricultural GDP ranged from 1.22% (min) to 3.06% (max), a clear long-term gain.

### 8. NumPy analysis

```python
years = spending_pct['year'].to_numpy()
values = spending_pct['value'].to_numpy()
```

`.to_numpy()` strips pandas' row labels and returns a raw numeric array — the format NumPy's math functions expect.

```python
yoy_change = np.diff(values)
```

`np.diff()` subtracts each value from the one before it, converting levels into year-on-year changes.

**Result:** Although individual years fluctuated as much as +0.79 or -0.57, the average yearly change was nearly zero (+0.047), indicating that the long-term increasing tendency was erratic rather than steady.

```python
values_30 = values[:30]
decades = values_30.reshape(3, 10)
```

`.reshape(3, 10)` reorganizes the first 30 years into a 3×10 grid, making it 3 decades of 10 years, which only works because the counts divide evenly.

**Result:** The decade averages — 1981-1990 (1.64%), 1991-2000 (2.23%), and 2001-2010 (2.14%) — demonstrate a significant increase from the 1980s to the 1990s before plateauing.

```python
correlation = np.corrcoef(values, fte_researchers)[0, 1]
```

`np.corrcoef()` returns a correlation matrix; `[0,1]` isolates the cross-correlation between the two real series.

**Result:** R&D spending intensity and researcher headcount did not rise together strongly, as seen by the modest positive correlation of **r = 0.24**, which suggests that spending increased more quickly than the size of the research workforce.

```python
merged = pd.merge(findex_total, asti_spending_yearly, on='year', how='inner')
```

`how='inner'` keeps only years present in both tables.

**Result:** The two datasets only shared one year (2014), with 2.78% R&D spending and 7.51% borrowed for farm/business. No coefficient was directly generated between the two datasets because a single point cannot sustain a legitimate correlation; this was documented as an honest data limitation rather than imposing a deceptive statistic.

### 9. Visualisation

Six charts were created, each corresponding to a particular rubric criterion and graphically verifying a previously computed number:

| Chart | Function | Purpose |
|---|---|---|
| Line chart | `plt.plot()` | 34-year R&D spending trend |
| Bar chart | `plt.bar()` | Decade average comparison |
| Grouped bar chart | `findex_pivot.plot(kind='bar')` | Male vs female borrowing, 2014 vs 2017 |
| Scatter plot | `plt.scatter()` | Spending vs researcher count (visualising r=0.24) |
| Box plot | `plt.boxplot()` | Spread/volatility of spending within each decade |
| Histogram | `plt.hist()` | Distribution of year-on-year changes |

**Result:** Each of the four chart types (bar, scatter, box, and histogram) required by the rubric were created, appropriately labeled with titles, axis labels, and colour, and each one visually supported a conclusion that had already been established numerically. For example, the scatter plot displayed a dispersed, non-diagonal cloud of points that matched the weak 0.24 correlation, and the box plot displayed 1981-1990 sitting lower with a tighter spread than the next two decades.

### 10. Database integration

```python
conn = sqlite3.connect('../db/project.db')
cursor = conn.cursor()
cursor.execute('CREATE TABLE IF NOT EXISTS findex (...)')
```

SQLite stores a full relational database in a single file, needing no separate server. `PRIMARY KEY AUTOINCREMENT` gives every row a unique ID, required for later targeted UPDATE/DELETE operations.

```python
findex_clean.to_sql('findex', conn, if_exists='replace', index=False)
```

Loads the cleaned DataFrame directly into the database table in one line.

```python
pd.read_sql_query("SELECT * FROM asti WHERE year >= 2010", conn)
```

Runs a real SQL query with a WHERE filter and loads the result back into pandas — demonstrating the "load database data into Pandas" requirement.

```python
cursor.execute("UPDATE asti SET indicator = '...' WHERE year = 2013 AND indicator = '...'")
cursor.execute("DELETE FROM findex WHERE year = 2017 AND sex = 'Total'")
```

Both are safe because of the WHERE clause; without it, every row in the table would be impacted. The impacted table was safely restored using `to_sql(..., if_exists='replace')` because the cleaned data was independently kept in the `findex_clean` DataFrame, even though a DELETE was executed during testing before its scope was fully determined. This is not an unsolved error; rather, it is documented as an example of safe recovery procedure.

```python
asti_from_db.to_csv('../data/processed/asti_clean.csv', index=False)
conn.close()
```

Completes the full pipeline loop: raw CSV → pandas → SQLite → SQL query → pandas → CSV export. `conn.close()` releases the database file.

**Result:** A functional `project.db` file with both tables, two clean CSV exports, and follow-up SELECT queries that demonstrate and validate each of the four CRUD processes (Create, Read, Update, and Delete).

### 11. Python/Excel analysis

```python
asti_wide = asti_clean.pivot_table(index='year', columns=['indicator','unit'], values='value').reset_index()
```

`.pivot_table()` reshapes long-format data (one row per measurement) into wide format (one row per year, one column per measure). Including `unit` alongside `indicator` in the pivot was a necessary correction — an earlier version without `unit` silently averaged incompatible measures (PPP dollars and percentages) into a single meaningless column; adding `unit` kept each measure in its own column.

```python
ws.conditional_formatting.add(f'{col_letter}2:{col_letter}35', ColorScaleRule(...))
```

Applies a red → yellow → green colour scale so low vs. high spending years are visible at a glance.

```python
chart = LineChart()
ws.add_chart(chart, "H2")
```

Embeds a native Excel chart object (not a pasted image) inside the workbook.

**Result:** `asti_analysis.xlsx` was produced and verified open in Excel, showing correctly colour-scaled cells (red for 1981's 1.22%, green for 2013's 3.06%) and an embedded line chart matching the matplotlib trend chart.

---

## Full Report

See `report.docx` / `report.pdf` for the complete 7-9 page write-up, including methodology, all visualisations, and conclusions.
