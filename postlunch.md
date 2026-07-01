---
title: "Census Data Analysis with Python Notebook"
teaching: 120 # teaching time in minutes
exercises: 120 # exercise time in minutes
---

:::::::::::::::::::::::::::::::::::::: questions
 
- How do you clean and prepare raw Census data for analysis?
- How do you rename columns, sort data, and compute summary statistics?
- What is data visualization and why does it matter for Census analysis?
- What makes a visualization effective versus misleading?
- Which Python tools are best for creating publication-ready plots?
::::::::::::::::::::::::::::::::::::::::::::::::
 
::::::::::::::::::::::::::::::::::::: objectives
 
- Clean a Census DataFrame: handle placeholder values, cast data types, and rename columns
- Sort and filter data to identify top geographic units
- Compute grouped summary statistics at the county and state level
- Define data visualization and explain its role in Census data analysis
- Recognize the principles of effective visualization design
- Identify common pitfalls (misleading charts, chartjunk, accessibility barriers)
- Use Python (Matplotlib, GeoPandas) to create choropleth maps, bar charts, and histograms
::::::::::::::::::::::::::::::::::::::::::::::::


---

## Importance of Data Cleaning 

Real-world data is messy. Before any analysis or visualization can happen, the data needs to be trustworthy — and that requires cleaning.

For Census data specifically, three problems show up almost every time:

- **Hidden missing values** - `NaN` means "no data" or null value. Left uncaught, it silently disrupts visualization
- **Wrong data types** -  the Census API returns everything as strings. Math on strings fails in Python
- **Unreadable column names** - `DP04_0058E` tells you nothing when looking at it first time, making it easy to mix up variables and hard for collaborators to follow your work

::::::::::::::::::::::::::::::::::::: callout
 
Data scientists typically spend **60–80% of project time** on data preparation — not analysis. The good news: for Census data, the cleaning steps are predictable and learnable.
 
::::::::::::::::::::::::::::::::::::::::::::::::

## Cleaning the Census Dataset
 
After downloading ACS data via the Census API (see the previous lesson), the raw DataFrame needs several cleaning steps before it is ready for analysis or visualization. This section walks through each step using the file you saved in Part 1.
 
### What We Are Working With ?
 
Jump to **Part 3** of the notebook. The data was downloaded as a CSV from the Census API and loaded into a pandas DataFrame. At this stage it has some rough edges:

- Every column is stored as a **string** — even numeric estimates like population counts
- Missing or suppressed values such as `NaN`
- Column names are raw variable codes like `DP04_0058E`, which are hard to read (optional)
- The dataset may have rows that should be excluded from analysis


:::::::::::::::::::::::::::::::::::::::::: challenge

### **Prerequisites**- Completion of Part 1 and 2 of the Notebook.  

Work through the interactive Python notebook Part `3` and `4` linked below, which covers everything on this page hands-on inside Google Colab. More explanation on the process of data cleaning explained below!
 
The hands-on work for this section:
 
- **Part 3: Data Cleaning** - null value removal, shapefile join, county ranking, summary statistics
- **Part 4: Visual Maps** - Bar charts, histogram, choropleth maps, and result interpretation

### <a href="https://colab.research.google.com/github/SpatialTurn/IGIC2026/blob/main/episodes/CensusDATA_Introduction.ipynb" target="_blank">Open the Notebook in Google Colab.</a>

::::::::::::::::::::::::::::::::::::::::::::::::::::::


### Step 1 — Cast Estimate Columns to Numbers
 
The Census API returns all values as strings. Before doing any math, convert estimate columns (those ending in `E`) to numeric:
 
```python
 
estimate_cols = [c for c in df.columns if c.endswith("E") and c not in ("NAME", "GEO_ID", "GEOID")] # excluding these columns
 
for col in estimate_cols:
    df[col] = pd.to_numeric(df[col], errors="coerce")
```
`errors="coerce"` turns anything that cannot be parsed (e.g., `"N"` for not applicable) into `NaN` automatically.


### Step 2 — Rename Columns to Human-Readable Labels
 
Raw ACS codes are hard to work with. Create a rename dictionary for the variables you downloaded:
 
```python
rename_map = {
    "DP04_0058E": "no_vehicle_households",
    "DP03_0062E": "median_household_income",
    "DP02_0001E": "total_households",
    # add more as needed
}
 
df.rename(columns=rename_map, inplace=True)
```

::::::::::::::::::::::::::::::::::::: callout
 
**Tip:** Keep a separate reference dictionary that maps the new names back to the original ACS codes and their full descriptions. This makes your work reproducible and easier to document.
 
```python
variable_reference = {
    "no_vehicle_households": ("DP04_0058E", "Occupied housing units with no vehicle available"),
    "median_household_income": ("DP03_0062E", "Median household income in the past 12 months"),
}
```
 
::::::::::::::::::::::::::::::::::::::::::::::::

### Step 3 — Drop or Flag Rows with Missing Data
 
Decide how to handle rows where your key variable is `NaN`:
 
```python
variable = "no_vehicle_households"
 
# Option A — drop rows missing the key variable entirely
df_clean = df.dropna(subset=[variable]).copy()
 
# Option B — flag them for inspection instead of dropping
df["data_missing"] = df[variable].isna()
print(df["data_missing"].value_counts())
```
 
Use Option A when you are ready to proceed to analysis. Use Option B while still exploring, so you can understand *why* values are missing (small population suppression, boundary changes, etc.).

### Step 4 — Sort the Data
 
Sorting makes it easy to find the highest and lowest values at a glance:
 
```python
# Top 10 geographic units by your variable
df_clean.sort_values(variable, ascending=False)[["NAME", "GEOID", variable]].head(10)
```
 
```python
# Bottom 10 (useful for spotting zeros or near-zero suppressed values)
df_clean.sort_values(variable, ascending=True)[["NAME", "GEOID", variable]].head(10)
```

### Step 5 — Summary Statistics
 
#### Single-variable summary
 
```python
print(df_clean[variable].describe().round(1))
```

This gives you count, mean, standard deviation, min, quartiles, and max — a fast sanity check before plotting.
 
#### Grouped by county
 
The first 5 characters of a tract-level `GEOID` are the state+county FIPS code. Use this to roll up tracts to the county level:
 
```python
df_clean["county_fips"] = df_clean["GEOID"].str[:5]
 
county_summary = (
    df_clean.groupby("county_fips")[variable]
    .agg(total="sum", average="mean", median="median", tract_count="count")
    .round(1)
    .sort_values("total", ascending=False)
)
 
print("Top 10 counties:")
display(county_summary.head(10))
```

#### Grouped by state
 
If your dataset spans multiple states, compare them side by side:
 
```python
df_clean["state_fips"] = df_clean["GEOID"].str[:2]
 
state_summary = (
    df_clean.groupby("state_fips")[variable]
    .agg(total="sum", average="mean", median="median", tracts="count")
    .round(1)
    .sort_values("total", ascending=False)
)
 
display(state_summary)
```

::::::::::::::::::::::::::::::::::::: keypoints
 
- Always cast Census columns to numeric before analysis — the API returns everything as strings
- Always check for missing data (`NaN`) to avoid visualization problems later on
- Rename cryptic variable codes to descriptive column names early in your workflow
- Use `groupby` with `.agg()` to compute multiple statistics at once across geographic units
::::::::::::::::::::::::::::::::::::::::::::::::


---
 
## Introduction to Data Visualization
 
### What Is Data Visualization?
 
Data visualization is the graphical representation of information. Instead of rows of numbers, it uses charts, maps, and diagrams to make patterns, trends, and outliers immediately understandable. For Census analysis specifically, visualization is what transforms a cleaned DataFrame into insight — showing *where* car-free households cluster, *which* counties are outliers, or *how* income varies across tracts.
 
There are two modes you will use throughout this workshop:
 
- **Exploratory visualization** — quick plots for *your own* understanding while cleaning and analyzing
- **Explanatory visualization** — polished charts and maps you share with others to communicate findings


### Why Visualization Matters for Census Data ?
 
Census datasets can have thousands of rows and dozens of columns. A 1,000-tract DataFrame is impossible to read directly. Visualization addresses this in a few key ways:
 
- A **choropleth map** shows the spatial distribution of an entire state's worth of tract-level data at once
- A **histogram** reveals whether values are evenly spread or heavily skewed toward a few areas
- A **bar chart** of top counties immediately answers "where is the problem concentrated?"
- **Scatter plots** uncover correlations between two variables (e.g., income vs. vehicle access) that summary statistics alone can miss

### Advantages and Risks
 
Visualization is powerful, but it can mislead as easily as it informs. Keep both sides in mind:
 
**Advantages:** 

- Spot trends in seconds
- Reduce cognitive load 
- Reveal outliers and clusters 
- Communicate across technical skill levels 
- Support storytelling with data
 
**Stuff to Avoid:**

- **Truncated axes** — starting a bar chart's y-axis at 500 instead of 0 can make a small difference look enormous
- **Chartjunk** — decorative elements like 3D effects, excessive gridlines, and gradient fills that add visual noise without adding information
- **Misleading color scales** — a diverging color palette centered at the wrong value distorts spatial patterns
- **Over-aggregation** — rolling tract-level data all the way up to state averages hides local variation


::::::::::::::::::::::::::::::::::::: callout
 
**Always ask:** Does this visualization show the whole picture, or only the part that supports a predetermined conclusion? Transparency about scale choices, data suppression, and margins of error is essential when sharing Census visualizations.
 
::::::::::::::::::::::::::::::::::::::::::::::::

### Principles of Effective Visualization
 
###### **Foundational Rules:**
 
1. **Choose the right chart type** — choropleth for spatial distribution, histogram for distribution shape, bar chart for ranking, scatter plot for relationships. Avoid pie charts for more than 4–5 categories.
2. **Label everything** — title, axis labels, units, and a legend. A chart with no axis labels cannot be interpreted.
3. **Be honest about scale** — never truncate axes without clearly disclosing it; clip outliers only after explaining why.
4. **Use colorblind-friendly palettes** — `viridis`, `YlOrRd`, and ColorBrewer palettes are designed to be perceptually uniform and accessible. Avoid raw red/green combinations.
5. **Remove what is not data** — maximize the ratio of information to ink. Every element should earn its place.
6. **Add accessibility** — include alt text for published figures; use patterns in addition to color where possible.

:::::::::::::::::::::::::::::::::::: challenge

Analyze U.S. Census population data for your assigned state and create a choropleth map
to visualize population patterns across census tracts. Then, determine the average tract
population and produce a second map that highlights which tracts fall above and below
this average

:::::::::::::::::::::::: solution

See the Solution to this Problem <a href="https://colab.research.google.com/github/SpatialTurn/IGIC2026/blob/main/episodes/Question2_AnswerKey.ipynb" target="_blank">Here.</a>

:::::::::::::::::::::::::::::::::
::::::::::::::::::::::::::::::::::::::::::::::::

::::::::::::::::::::::::::::::::::::: challenge

In Part 4 of the Notebook. Complete the following:
 
1. Run the basic choropleth (Section 4.2) using the default `viridis` colormap
2. Switch the colormap in Section 4.3 to `Blues` and observe how the interpretation changes
3. In the bar chart (Section 4.4), change `head(15)` to `head(10)` and add county names instead of FIPS codes by joining with a county name lookup
4. In the histogram (Section 4.5), describe in one sentence what the shape of the distribution tells you about how your variable is distributed across tracts

::::::::::::::::::::::::::::::::::::::::::::::::


::::::::::::::::::::::::::::::::::::: keypoints
 
- Exploratory plots help you understand your data; explanatory plots help others understand your findings
- Choropleth maps, histograms, and bar charts each answer a different question about Census data
- Color scale choices, axis ranges, and aggregation level all affect how a visualization is interpreted
- Use colorblind-friendly palettes and always label axes, titles, and legends
- Transparency about data suppression and margins of error is an ethical requirement when publishing Census visualizations
::::::::::::::::::::::::::::::::::::::::::::::::