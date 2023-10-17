# deduplication-stats-questdb
Just some scripts to play with event deduplication and QuestDB

## some details about the experiment

I am running this test on an AWS EC2 instance (m6a.4xlarge, 16 CPUs, 64 Gigs of RAM, GP3 EBS volume). I will be ingesting 15 uncompressed CSV files, each containing 12,614,400 rows, for a total of 189,216,000 rows representing 12 years of hourly data. The data represents synthetic ecommerce statistics, with one hourly entry per country (ES, DE, FR, IT, and UK) and category (WOMEN, MEN, KIDS, HOME, KITCHEN).

The dataset I built for this experiment can be found at https://archive.org/details/questdb-csv-dedup-demo-files.

Data will be ingested into five tables (one per country) with this structure:

Structure for QuestDB

```
CREATE TABLE IF NOT EXISTS  'ecommerce_sample_test_{country}' (
                            ts TIMESTAMP,
                            country SYMBOL capacity 256 CACHE,
                            category SYMBOL capacity 256 CACHE,
                            visits LONG,
                            unique_visitors LONG,
                            avg_unit_price DOUBLE,
                            sales DOUBLE
                            ) timestamp (ts) PARTITION BY DAY WAL DEDUP UPSERT KEYS(ts,country,category);
```

Structure for ClickHouse
```
CREATE TABLE IF NOT EXISTS  ecommerce_sample_test_{country} (
                        ts datetime,
                        country enum('UK'=1, 'DE'=2, 'FR'=3, 'IT'=4, 'ES'=5),
                        category enum('WOMEN'=1, 'MEN'=2, 'KIDS'=3, 'HOME'=4, 'KITCHEN'=5, 'BATHROOM'=6),
                        visits UInt32,
                        unique_visitors UInt32,
                        avg_unit_price Decimal32(4),
                        sales  Decimal64(4)
                        ) ENGINE = ReplacingMergeTree
                        PRIMARY KEY(ts, country, category);
```

Structure for Timescale
```
cur.execute("""
                        CREATE TABLE IF NOT EXISTS  ecommerce_sample_{country} (
                        ts TIMESTAMPTZ,
                        country TEXT,
                        category TEXT,
                        visits INT,
                        unique_visitors INT,
                        avg_unit_price DOUBLE PRECISION NULL,
                        sales DOUBLE  PRECISION NULL,
                        UNIQUE (ts, country, category)
                        );

                        """
                        )
            cur.execute("""
                    CREATE UNIQUE INDEX IF NOT EXISTS ecommerce_sample_{country}_unique_idx ON ecommerce_sample_test(ts,country, category);
                        """)
            cur.execute("""
                    SELECT create_hypertable('ecommerce_sample_{country}_test', 'ts', if_not_exists => TRUE);
                        """)
            cur.execute("""
                    CREATE INDEX IF NOT EXISTS ecommerce_sample_{country}_idx ON ecommerce_sample_test(ts,country, category);

                        """)
```

The total size of the raw CSVs is about 17Gig and I am reading from a RAM disk to minimise the impact of reading the files. I am reading/parsing/ingesting from up to 8 files in parallel. The scripts are written in Python, so very likely we could optimise ingestion a bit by reducing CSV parsing time using a different language, but this is not a benchmark, we just want a ballpark of the impact of DEDUP on ingestion.
The dataset I created for this experiment is available at https://archive.org/details/questdb-csv-dedup-demo-files, and the scripts can be found at https://github.com/javier/deduplication-stats-questdb.
