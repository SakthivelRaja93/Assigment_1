# Mini Project 1: SecureCheck

##Clean the data

from pathlib import Path

import pandas as pd


INPUT = Path("data/traffic_stops.csv")
OUTPUT = Path("data/traffic_stops_clean.csv")
REQUIRED = [
    "stop_date", "stop_time", "country_name", "driver_gender", "driver_age",
    "driver_race", "violation", "search_conducted", "search_type",
    "stop_outcome", "is_arrested", "stop_duration", "drugs_related_stop",
]
FLAG_COLUMNS = ["search_conducted", "is_arrested", "drugs_related_stop"]
TRUE_FALSE = {"true": 1, "1": 1, "yes": 1, "false": 0, "0": 0, "no": 0}


def clean_traffic_stops(input_path=INPUT, output_path=OUTPUT):
    frame = pd.read_csv(input_path)
    frame.columns = frame.columns.str.strip().str.lower()
    frame = frame.dropna(axis="columns", how="all")

    if "driver_age" not in frame and "driver_age_raw" in frame:
        frame["driver_age"] = frame["driver_age_raw"]
    missing = [column for column in REQUIRED if column not in frame]
    if missing:
        raise ValueError(f"Dataset is missing required columns: {missing}")

    frame = frame[REQUIRED].copy()
    frame["stop_date"] = pd.to_datetime(frame["stop_date"], errors="coerce")
    frame["driver_age"] = pd.to_numeric(frame["driver_age"], errors="coerce")
    frame.loc[~frame["driver_age"].between(15, 100), "driver_age"] = pd.NA

    for column in FLAG_COLUMNS:
        values = frame[column].astype(str).str.strip().str.lower()
        frame[column] = values.map(TRUE_FALSE).astype("Int64")

    for column in ("country_name", "violation", "stop_outcome"):
        frame[column] = frame[column].astype("string").str.strip()

    frame = frame.dropna(subset=["stop_date", "stop_time", "country_name", "violation"])
    frame["stop_hour"] = pd.to_datetime(frame["stop_time"], errors="coerce").dt.hour
    frame = frame.drop_duplicates().reset_index(drop=True)
    frame.to_csv(output_path, index=False)
    print(f"Saved {len(frame)} cleaned records to {output_path}")
    return frame


if __name__ == "__main__":
    clean_traffic_stops()


##: Create the SQL database

```sql
CREATE TABLE IF NOT EXISTS traffic_stops (
    id INTEGER PRIMARY KEY,
    stop_date TEXT NOT NULL,
    stop_time TEXT,
    stop_hour INTEGER,
    country_name TEXT,
    driver_gender TEXT,
    driver_age INTEGER,
    driver_race TEXT,
    violation TEXT,
    search_conducted INTEGER CHECK (search_conducted IN (0, 1) OR search_conducted IS NULL),
    search_type TEXT,
    stop_outcome TEXT,
    is_arrested INTEGER CHECK (is_arrested IN (0, 1) OR is_arrested IS NULL),
    stop_duration TEXT,
    drugs_related_stop INTEGER CHECK (drugs_related_stop IN (0, 1) OR drugs_related_stop IS NULL)
);

CREATE INDEX IF NOT EXISTS idx_stops_date ON traffic_stops(stop_date);
CREATE INDEX IF NOT EXISTS idx_stops_country_violation ON traffic_stops(country_name, violation);
```

Create `load_data.py`:

```python
import sqlite3

import pandas as pd


with open("schema.sql", encoding="utf-8") as schema_file:
    schema = schema_file.read()

frame = pd.read_csv("data/traffic_stops_clean.csv")
with sqlite3.connect("traffic_stops.db") as connection:
    connection.execute("DROP TABLE IF EXISTS traffic_stops")
    connection.executescript(schema)
    frame.to_sql("traffic_stops", connection, if_exists="append", index=False)

print(f"Loaded {len(frame)} rows into traffic_stops.db")
```
##: Add the brief's SQL analyses

```sql
-- Most common violations
SELECT violation, COUNT(*) AS stops
FROM traffic_stops
GROUP BY violation
ORDER BY stops DESC;

-- Search rate and arrest rate by violation
SELECT violation,
       COUNT(*) AS stops,
       AVG(search_conducted) AS search_rate,
       AVG(is_arrested) AS arrest_rate
FROM traffic_stops
GROUP BY violation
ORDER BY stops DESC;

-- Arrest rate by driver age group
SELECT CASE
         WHEN driver_age < 25 THEN 'Under 25'
         WHEN driver_age < 40 THEN '25-39'
         WHEN driver_age < 60 THEN '40-59'
         ELSE '60+'
       END AS age_group,
       COUNT(*) AS stops,
       AVG(is_arrested) AS arrest_rate
FROM traffic_stops
WHERE driver_age IS NOT NULL
GROUP BY age_group
ORDER BY arrest_rate DESC;

-- Stops by hour of day
SELECT stop_hour, COUNT(*) AS stops
FROM traffic_stops
WHERE stop_hour IS NOT NULL
GROUP BY stop_hour
ORDER BY stop_hour;

-- Drug-related stop rate by country
SELECT country_name, COUNT(*) AS stops, AVG(drugs_related_stop) AS drug_stop_rate
FROM traffic_stops
GROUP BY country_name
ORDER BY drug_stop_rate DESC;

-- Duration by violation
SELECT violation, stop_duration, COUNT(*) AS stops
FROM traffic_stops
GROUP BY violation, stop_duration
ORDER BY violation, stops DESC;
```
##: Build a searchable Streamlit dashboard

Create `app.py`:

```python
import sqlite3

import pandas as pd
import streamlit as st


st.set_page_config(page_title="SecureCheck", layout="wide")
st.title("SecureCheck: Traffic Stop Analysis")
st.caption("Educational analysis of the supplied, de-identified course dataset.")


@st.cache_data
def load_options():
    with sqlite3.connect("traffic_stops.db") as connection:
        countries = pd.read_sql_query(
            "SELECT DISTINCT country_name FROM traffic_stops ORDER BY country_name", connection
        )["country_name"].dropna().tolist()
        violations = pd.read_sql_query(
            "SELECT DISTINCT violation FROM traffic_stops ORDER BY violation", connection
        )["violation"].dropna().tolist()
    return countries, violations


countries, violations = load_options()
country = st.selectbox("Country", ["All"] + countries)
violation = st.selectbox("Violation", ["All"] + violations)

conditions = []
parameters = []
if country != "All":
    conditions.append("country_name = ?")
    parameters.append(country)
if violation != "All":
    conditions.append("violation = ?")
    parameters.append(violation)
where_sql = " WHERE " + " AND ".join(conditions) if conditions else ""

with sqlite3.connect("traffic_stops.db") as connection:
    summary = pd.read_sql_query(
        "SELECT COUNT(*) AS stops, AVG(search_conducted) AS search_rate, "
        "AVG(is_arrested) AS arrest_rate FROM traffic_stops" + where_sql,
        connection,
        params=parameters,
    ).iloc[0]
    by_violation = pd.read_sql_query(
        "SELECT violation, COUNT(*) AS stops FROM traffic_stops" + where_sql
        + " GROUP BY violation ORDER BY stops DESC",
        connection,
        params=parameters,
    )
    hour_condition = " AND " if where_sql else " WHERE "
    hour_query = (
        "SELECT stop_hour, COUNT(*) AS stops FROM traffic_stops"
        + where_sql
        + hour_condition
        + "stop_hour IS NOT NULL GROUP BY stop_hour ORDER BY stop_hour"
    )
    by_hour = pd.read_sql_query(hour_query, connection, params=parameters)

metric1, metric2, metric3 = st.columns(3)
metric1.metric("Stops", f"{int(summary['stops']):,}")
metric2.metric("Search rate", f"{summary['search_rate']:.1%}" if pd.notna(summary['search_rate']) else "N/A")
metric3.metric("Arrest rate", f"{summary['arrest_rate']:.1%}" if pd.notna(summary['arrest_rate']) else "N/A")
st.subheader("Stops by violation")
st.bar_chart(by_violation.set_index("violation"))
st.subheader("Stops by hour")
if not by_hour.empty:
    st.line_chart(by_hour.set_index("stop_hour"))
else:
    st.info("No stop-hour values are available for this filter.")
```