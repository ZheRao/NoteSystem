# Imports & Setup

## Important Components
- `builder = pyspark.sql.SparkSession.builder.appName().master()`
- `builder = builder.config(name, value)`
- `spark = builder.getOrCreate()`
- `spark.sparkContext.setLogLevel()`


```python
# src/qbo_etl/utils/spark.py

from __future__ import annotations

from typing import Mapping

from pyspark.sql import SparkSession


DEFAULT_SPARK_CONFIG: dict[str, str] = {
    # correctness / reproducibility
    "spark.sql.session.timeZone": "America/Regina",

    # local developer ergonomics
    "spark.ui.showConsoleProgress": "true",

    # local performance defaults
    "spark.sql.shuffle.partitions": "8",

    # resource defaults
    "spark.driver.memory": "4g",
    
    # UI
    "spark.ui.enabled", "true",
    "spark.ui.showConsoleProgress", "true"
}


def get_spark(
    app_name: str = "qbo-etl-local",
    master: str = "local[*]",
    extra_conf: Mapping[str, str] | None = None,
    log_level: str = "WARN",
) -> SparkSession:
    """
    Create or retrieve a SparkSession using standardized project defaults.

    Parameters
    ----------
    app_name:
        Name shown in Spark UI / logs.
    master:
        Spark master URL, e.g. 'local[*]' for all local cores.
    extra_conf:
        Optional config overrides/additions.
    log_level:
        SparkContext log level, e.g. INFO, WARN, ERROR.

    Returns
    -------
    SparkSession
    """
    conf = dict(DEFAULT_SPARK_CONFIG)
    if extra_conf:
        conf.update({k: str(v) for k, v in extra_conf.items()})

    builder = (
        SparkSession.builder
        .appName(app_name)
        .master(master)
    )

    for key, value in conf.items():
        builder = builder.config(key, value)

    spark = builder.getOrCreate()
    spark.sparkContext.setLogLevel(log_level)

    return spark
```

## Details

### Mental Model

1. **Identity** — app name, local vs cluster master
2. **Resources** — driver memory, shuffle partitions
3. **Correctness defaults** — timezone, ANSI mode, legacy behavior if needed
4. **Developer ergonomics** — console progress, log level, UI visibility

### Common Options

1) `master`  
Controls where Spark runs. For local dev: `local[*]`. Official docs list examples like `local`, `local[4]`, and remote masters such as `spark://....`

2) `appName`  
This shows up in the Spark UI and logs, so make it descriptive, like `qbo-etl-local` or `qbo-etl-bronze-load`.

3) `spark.driver.memory`  
One of the most important knobs locally. The Spark configuration docs list driver memory as a standard property, and insufficient heap is a common cause of failures.

4) `spark.sql.shuffle.partitions`  
This controls the default number of shuffle partitions for Spark SQL operations. For local development, the production default is often too high, 
so reducing it usually makes local ETL faster and less noisy. This is a standard Spark SQL config documented in Spark configuration.

5) `spark.sql.session.timeZone`  
This one matters a lot for finance and timestamp-heavy ETL. Spark uses the session time zone when interpreting/displaying timestamps, 
and if it is not explicitly set, behavior can depend on the JVM/system local timezone. Official docs explicitly describe that timestamp handling 
depends on `spark.sql.session.timeZone`.  

6) `spark.ui.showConsoleProgress`  
Useful in development because it gives visible job progress in the console. It is part of Spark’s config system.

7) `spark.sql.ansi.enabled`  
Potentially valuable if you want stricter SQL semantics and earlier failure instead of permissive weirdness. Spark has been improving ANSI compliance across recent versions.






# Inspect

Inspect active config

```python
for k, v in spark.sparkContext.getConf().getAll():
    print(k, v)
```

Inspect UI address for the current session

```python
print("Spark UI:", spark.sparkContext.uiWebUrl)
```


# Stop

Stop spark session and release resources

```python
spark.stop()
```






# Logging
