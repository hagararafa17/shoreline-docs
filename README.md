# Shoreline Analytics: Alexandria coastal erosion database

PostGIS database and documentation for the DEPI graduation project **Satellite-Based Coastal Erosion & Risk Mapping in Alexandria**. It stores nine years (2018–2026) of Sentinel-2 shorelines, the rate at which the shoreline moves along 145 transects, a **Risk Class** for each transect and a **Confidence Score** that says how far to trust it.

> **Status: provisional.** Independent validation of the shorelines is still pending. Every result here is an apparent trend, not a surveyed fact. See [Limitations](#limitations).

**Live documentation:** https://hegererefe111-debug.github.io/shoreline-docs/

## Documentation pages

| Page | What it shows |
|---|---|
| Overview | The numbers, the risk split and the limits |
| ER | The seven tables and their links |
| DFD | How data moves from the satellite to the dashboard |
| Activity | The loading and risk steps, lane by lane |
| Logical | Entities, attributes, keys and rules |
| Physical | The real SQL, with the run order |
| Team | Who owns which task and document |

## Repository layout

```
index.html, overview.html, er.html, dfd.html,
activity.html, logical.html, physical.html, team.html   the documentation site (GitHub Pages)
database/
  sql/        01_create_tables.sql  02_views.sql  03_analysis_tables.sql  04_risk_confidence.sql
  scripts/    load_shorelines.py    load_transects.py
  requirements.txt
```

## Database design

PostgreSQL 17 with PostGIS 3.5, schema `shoreline`, every geometry in EPSG:32636 (WGS 84 / UTM zone 36N).

| Table | Rows | Holds |
|---|---|---|
| `sectors` | 3 | The three study zones: Corniche Infill, Eastern Harbour, Montaza Bay |
| `shoreline_segments` | 205 | One shoreline line per sector, year and segment |
| `transect_rates` | 145 | A rate in metres per year for each transect, with R², p-values and reliability |
| `cross_sensor_validation` | 3 | Sentinel-2 against Landsat 9 for 2026, per sector |
| `risk_thresholds` | 5 | The five risk classes and their limits |
| `confidence_parameters` | 14 | Weights and settings of the Confidence Score |
| `risk_assessments` | 145 | One Risk Class and one Confidence Score per transect |

Views: `v_sector_year_summary`, `v_risk_calc`, `v_transect_risk`, `v_sector_risk`. Function: `shoreline.assess_risk()`.

## Setup

The files must run in this order. Run the SQL files in pgAdmin's Query Tool, or with `psql -f`.

1. `database/sql/01_create_tables.sql` (drops and re-creates every table)
2. `python database/scripts/load_shorelines.py` (3 sectors and 205 segments, one transaction)
3. `database/sql/02_views.sql`
4. `database/sql/03_analysis_tables.sql`
5. `database/sql/04_risk_confidence.sql`
6. `python database/scripts/load_transects.py` (145 transects and the Landsat check, then `assess_risk()`)

Both scripts ask for the database password and never store it. They load inside one transaction and check the row counts before committing, so a wrong file saves nothing.

```
pip install -r database/requirements.txt
```

The scripts read a data folder (default `D:\shoreline_project\data\Alexandria_Shoreline_v2`, or pass another path as the first argument) that holds:

```
sectors.gpkg
shorelines_by_sector_ANALYSIS_2018_2026_v2.gpkg
transect_statistical_reliability.csv
erosion_rate_sensitivity_test.csv
Significant_Transect_Spatial_Verification.csv
Independent_Cross_Sensor_Validation_2026.csv
```

The raw data is not stored in this repository.

## Risk and confidence

**Risk Class** describes the size and direction of change. It comes from the transect rate and the USGS ranking in `risk_thresholds`:

| Class | Rate (m per year) |
|---|---|
| VERY_LOW | above +2 (accretion) |
| LOW | +1 to +2 |
| MODERATE | −1 to +1 |
| HIGH | −2 to −1 |
| VERY_HIGH | below −2 (erosion) |

**Confidence Score** (0 to 1) describes the strength of the evidence. It is a weighted average of six parts: statistical evidence from the adjusted p-value (0.30), R² (0.15), observed years out of nine (0.15), stability of the rate without the 2024 scene (0.15), shoreline line quality of the sector (0.10) and Landsat 9 agreement of the sector (0.15). The band is HIGH from 0.80, MEDIUM from 0.60, otherwise LOW, and **HIGH is only given to transects with a significant trend**.

The weights and limits are this project's proposal, built on the components the remote-sensing team listed. They live in `confidence_parameters`, so changing one is an `UPDATE` followed by `SELECT shoreline.assess_risk();`.

Risk and confidence are kept apart on purpose: a transect can show strong erosion with low confidence.

## Results at a glance

- The median transect moves slightly seaward in every sector (+0.21 to +0.93 m per year).
- Four transects in Montaza Bay show accretion that survives the multiple-testing correction. They are local, not a trend of the whole bay.
- 13 transects fall in the HIGH or VERY_HIGH class. None has a HIGH Confidence Score, so they are places to check, not findings.

## Limitations

- All shorelines and rates are provisional; independent validation is pending.
- The shoreline is a water-mask proxy at 10 m resolution, not a surveyed line.
- Landsat 9 was compared for 2026 only.
- Most transects fit a straight line poorly (median R² 0.13); only 4 of 145 pass the corrected significance test.
- The 2024 scene is an alternative acquisition; its rate effect is tested in `rate_excl_2024`.
- The Confidence Score weights are a proposal and can be changed.

## Team

Asmaa Bahaa (data ingestion), Doha Yehia (shoreline extraction and transect analysis), Hagar Arafa (PostGIS database), Nourhan Fathy (dashboard), Shahd Abdelnasser (API and deployment). Roles are on the Team page.

## Data

Sentinel-2 L2A imagery from Microsoft Planetary Computer. Landsat 9 Collection 2 Level-2 for the 2026 cross-check.
