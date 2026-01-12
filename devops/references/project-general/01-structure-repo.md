# Config & State

```text

config/
  README.md

  env/                      # “what this run is” (dev/stage/prod, local, etc.)
    dev.json
    prod.json

  io/                       # “where stuff is”
    paths.local.json
    paths.wsl.json
    paths.prod.json

  domain/                   # “business decisions / logic”
    companies.json          # canonical company list + ids
    mappings/
      account_map.json
      class_map.json
      location_map.json
    rules/
      exclusions.json       # ignore accounts/classes/vendors/etc.
      normalization.json    # rename rules, standardization, etc.

  runtime/                  # knobs that change behavior but aren’t “business logic”
    logging.json
    spark.json
    performance.json

state/                      # machine-written (gitignored)
  pipeline_meta.json        # last_load_date, watermark, last_fx_rate_date, etc.
  checkpoints/
  locks/


```