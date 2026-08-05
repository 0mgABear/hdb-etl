# HDB Resale Flat Prices - ETL Pipeline & Architecture

**Part 1**: Python/pandas ETL pipeline that cleans and transforms HDB resale flat price data

**Part 2**: AWS Architecture for data ingestion and Exploitation

## Part 1 - ETL

The pipeline `etl.ipynb` extracts the required datasets from the data.gov.sg API. It then combines the data into 1 master dataset, profiles the dataset and then validation and cleaning rules are applied.

### Setup

Requires Python 3.12 (to be compatible with ydata-profiling)

```
# create and activate a virtual environment
python3.12 -m venv venv
source venv/bin/activate        # macOS/Linux

# install dependencies
pip install -r requirements.txt

```

### Running

Open `etl.ipynb` in Jupyter Notebook (Or VS Code) and run all cells sequentially.

```bash
jupyter notebook etl.ipynb

```

The notebook is ordered as a sequential pipeline. A clean **Restart Kernel → Run All** reproduces every output from scratch, including downloading the raw data.

### Outputs

All outputs are written under `data/`:

| Folder              | Contents                                                                              |
| ------------------- | ------------------------------------------------------------------------------------- |
| `data/raw/`         | Raw source CSVs, as downloaded (unmodified)                                           |
| `data/cleaned/`     | Records passing all data-quality rules (anomalies retained, flagged)                  |
| `data/transformed/` | Cleaned data after Resale Identifier build + identifier deduplication                 |
| `data/hashed/`      | Transformed data with the SHA-256 hashed identifier column                            |
| `data/failed/`      | Rejected records (validation failures + duplicates), each tagged with a `fail_reason` |
| `data/anomalies/`   | Price outliers, flagged for review (not rejected)                                     |

Profiling reports (`data/profiling_report*.html`) are also generated — open in a browser to view.

Pipeline stages
Extract — query the data.gov.sg collection, select datasets covering 2012–2016, download.
Combine — concatenate into a single master dataset.
Profile — ydata-profiling + targeted manual profiling to establish the data's properties.
Scope — filter to the 2012–2016 assessment window (profiling revealed out-of-range rows).
Validate — rules for month, town, flat_type, flat_model, storey_range; failures routed to failed with reasons.
Recompute lease — remaining lease as of today, rounded down to years and months.
Deduplicate — on the composite key (all source columns except resale_price); keep higher price.
Detect anomalies — contextual IQR outliers within town + flat_type; flagged, not deleted.
Transform — build the Resale Identifier, deduplicate on it, hash with SHA-256.
Output — write all datasets.

## Part 2

Two AWS solution architectures are provided as PNG diagrams (see `architecture/`):

- **Ingestion** — batch pull from data.gov.sg into a private VPC, landing in S3, transformed by Glue.
- **Exploitation** — Tableau on EC2 querying the data via Athena, with traffic kept private through PrivateLink and VPC endpoints.

## Key Assumptions

### Assumptions

### Part 1

- **Lease commencement year only.** The dataset only gives the year a lease started, not the month or day. I used Jan 1 as the start date. I compared Jan against Dec and the difference was always exactly 11 months, so the choice doesn't move the numbers much either way. Jan 1 gives the lower of the two, so if it's wrong it under-states the remaining lease rather than over-stating it.

### Part 2

- **Data volumes will grow beyond current file sizes.** Current files are under 30MB, but the design targets the >100MB requirement stated in the brief.
- **Analysts reach Tableau over a private network** Analysts connect to Tableau over the company's internal network, not the public internet. The brief doesn't say how users reach Tableau Server, so I assumed it stays private.
- **HDB's platform runs in a single VPC**, with separate subnets for ingestion and analytics.
