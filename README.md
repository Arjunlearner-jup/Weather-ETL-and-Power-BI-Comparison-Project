# Weather ETL and Power BI Comparison Project

This project builds a small end-to-end weather data pipeline that collects forecast data from the Open-Meteo API, stores it in MongoDB, loads it into PostgreSQL, reshapes the source CSV file, and creates a final comparison dataset for visualization in Power BI.

## Project workflow

The pipeline is organized into four Python scripts, each handling one stage of the process.

1. `01extractjson.py` fetches hourly weather forecast data from the Open-Meteo API and stores one document per forecast hour in MongoDB.
2. `02ETL.py` creates the PostgreSQL database if needed, loads records from MongoDB into PostgreSQL, and exports the PostgreSQL table to `raw_forecasts.csv`.
3. `03strucsv.py` cleans the raw Open-Meteo CSV file by extracting metadata and restructuring the hourly data into a flat CSV called `open-meteo-59.91N10.74E9m_cleaned.csv`.
4. `04final_merge.py` compares the PostgreSQL export and the cleaned CSV row by row, then writes the result to `final_weather_comparison.csv`.

## Files

| File | Purpose |
|---|---|
| `01extractjson.py` | Extracts forecast JSON from Open-Meteo and inserts flattened records into MongoDB. |
| `02ETL.py` | Moves weather data from MongoDB to PostgreSQL and exports PostgreSQL data to CSV. |
| `03strucsv.py` | Cleans and reshapes the raw Open-Meteo CSV source file. |
| `04final_merge.py` | Produces a side-by-side comparison dataset for PostgreSQL and CSV values. |

## Data flow

The data flow starts with the Open-Meteo API and ends in Power BI as a comparison report.

```text
Open-Meteo API
   -> MongoDB (raw_forecasts)
   -> PostgreSQL (raw_forecasts table)
   -> raw_forecasts.csv

Open-Meteo raw CSV
   -> open-meteo-59.91N10.74E9m_cleaned.csv

raw_forecasts.csv + cleaned CSV
   -> final_weather_comparison.csv
   -> Power BI dashboard
```

## Script details

### 1. Extract JSON to MongoDB

`01extractjson.py` sends a request to the Open-Meteo forecast API for hourly `temperature_2m` data, then stores the response in MongoDB as flattened hourly records instead of one nested JSON document. Each stored document includes `forecast_time`, `temperature_2m`, `latitude`, `longitude`, `elevation`, `timezone`, and `ingested_at`, which makes downstream querying and loading easier [file:444].

### 2. ETL from MongoDB to PostgreSQL

`02ETL.py` checks whether the PostgreSQL database `weather_123` exists and creates it if necessary. It then creates the `raw_forecasts` table, reads up to 72 documents from MongoDB, inserts them into PostgreSQL with `forecast_time` marked as unique, and exports the final table to `raw_forecasts.csv`.

### 3. Structure the source CSV

`03strucsv.py` reads the raw Open-Meteo CSV file, extracts metadata such as `latitude`, `longitude`, and `elevation`, then locates the actual hourly table starting at the `time` column header. It splits the timestamp into separate `date` and `time` fields, standardizes the temperature column name to `temperature_2m`, adds location columns to every row, and saves the cleaned result to `open-meteo-59.91N10.74E9m_cleaned.csv`.

### 4. Final merge for comparison

`04final_merge.py` loads the PostgreSQL CSV export and the cleaned Open-Meteo CSV into pandas DataFrames, standardizes column names, and converts the date and time fields into matching formats. It then renames the weather and location columns with `_psql` and `_csv` suffixes, adds row IDs so both datasets align row by row, merges them, and saves the output as `final_weather_comparison.csv`.

## Output datasets

The project creates three main CSV outputs.

- `raw_forecasts.csv`: Export of the PostgreSQL `raw_forecasts` table.
- `open-meteo-59.91N10.74E9m_cleaned.csv`: Cleaned version of the raw Open-Meteo CSV file.
- `final_weather_comparison.csv`: Final side-by-side comparison dataset used in Power BI.

## Power BI report

The Power BI report uses the final comparison dataset to visualize PostgreSQL and CSV temperatures, compare trends across time, and calculate temperature difference measures such as `Temp Diff = [PSQL Temp] - [CSV Temp]`. The report is designed to show whether the PostgreSQL-loaded data matches the structured CSV data row by row.

## Requirements

To run this project locally, the following tools and libraries are needed:

- Python 3.x
- MongoDB running locally on `mongodb://localhost:27017/` 
- PostgreSQL running locally on port `5432` 
- Python packages: `requests`, `pymongo`, `psycopg2`, and `pandas` 
- Power BI Desktop for the final dashboard

Install the Python dependencies with:

```bash
pip install requests pymongo psycopg2 pandas
```

## How to run

Run the scripts in this order to reproduce the pipeline [file:444][file:445][file:446][file:447]:

```bash
python 01extractjson.py
python 02ETL.py
python 03strucsv.py
python 04final_merge.py
```

Then import `final_weather_comparison.csv` into Power BI to build the comparison report [file:447][file:448].

## Notes

The PostgreSQL load script limits the import to 72 MongoDB documents, so the comparison is based on that subset rather than the entire MongoDB collection. The final merge aligns rows by generated `row_id`, which means the comparison is positional rather than based on a natural join key such as date-time equality.
