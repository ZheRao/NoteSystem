# `devops` — Engineering Notes, Mechanisms, and System Invariants

A structured engineering knowledge base for capturing:

* reusable mechanisms
* architectural decisions
* development principles
* operational references
* system design patterns
* implementation cautions
* debugging insights

The goal of this repository is designed as a **personal engineering operating system**:

## Philosophy

This repository separates knowledge into two major layers:

| Layer          | Purpose                                                             |
| -------------- | ------------------------------------------------------------------- |
| `foundations/` | Deep understanding, architecture, invariants, mechanisms, decisions |
| `references/`  | Quick syntax lookup, operational reminders, practical commands      |

## Foundations

`foundations/` contains durable engineering understanding.

They explain:

* why systems behave the way they do
* tradeoffs
* failure modes
* architectural consequences
* operational cautions
* scaling implications

### Foundations → Mechanisms

Low-level mechanics and operational understanding.

These documents explain:

* how systems actually work
* execution timelines
* lifecycle behavior
* runtime implications
* hidden edge cases


#### Data Ecosystem

* [01-cluless-intro](./foundations/mechanisms/data-ecosystem/01-cluless-intro.md)


#### IO

* [01-atomic-write](./foundations/mechanisms/io/01-atomic-write.md)
* [02-atomic-write-df](./foundations/mechanisms/io/02-atomic-write-df.md)
* [03-serialization](./foundations/mechanisms/io/03-serialization.md)
* [04-read-write](./foundations/mechanisms/io/04-read-write.md)
* [05-path](./foundations/mechanisms/python/05-path.md)


#### OS

* [01-bash-file-transfer-win](./foundations/mechanisms/os/01-bash-file-transfer-win.md)


#### Pandas

* [01-groupby](./foundations/mechanisms/pandas/01-groupby.md)
* [02-file-system](./foundations/mechanisms/pandas/02-file-system.md)
* [03-pandas-series](./foundations/mechanisms/pandas/03-pandas-series.md)
* [04-memory-lifecycle](./foundations/mechanisms/pandas/04-memory-lifecycle.md)
* [05-io](./foundations/mechanisms/pandas/05-io.md)


#### Python

* [01-import-and-modules](./foundations/mechanisms/python/01-import-and-modules.md)
* [02-dataclass](./foundations/mechanisms/python/02-dataclass.md)
* [03-requests](./foundations/mechanisms/python/03-requests.md)
* [04-errors](./foundations/mechanisms/python/04-errors.md)
* [05-python-path](./foundations/mechanisms/python/05-path.md)
* [06-data-objects](./foundations/mechanisms/python/06-data-structure.md)


#### Spark

* [00-gentle-intro](./foundations/mechanisms/spark/00-gentle-intro.md)
* [01-core-orchestration](./foundations/mechanisms/spark/01-core-orchestration.md)
* [02-spark-exe-timeline](./foundations/mechanisms/spark/02-spark-exe-timeline.md)
* [03-mechanics-and-cautions](./foundations/mechanisms/spark/03-mechanics-and-cautions.md)
* [04-schema-handling](./foundations/mechanisms/spark/04-schema-handling.md)
* [05-spark-context](./foundations/mechanisms/spark/05-spark-context.md)
* [06-file-system](./foundations/mechanisms/spark/06-file-system.md)
* [07-config](./foundations/mechanisms/spark/07-config.md)
* [08-data-ops](./foundations/mechanisms/spark/08-data-ops.md)

#### Beyond Data Infra/System

* [00-intuition](./foundations//mechanisms/beyond-infra/00-intuition.md)
* **Serving Layer**
    * [01-SQLite](./foundations/mechanisms/beyond-infra/serving-layer/01-sqlite.md)
    * [02-FastAPI](./foundations//mechanisms/beyond-infra/serving-layer/02-fastapi.md)

#### Vendor Web Design

* [01-web-export-data](./foundations/mechanisms/vendor-web-design/01-web-export-data.md)
* [02-reverse-engineering-playbook](./foundations/mechanisms/vendor-web-design/02-reverse-engineering-playbook.md)
* [03-graphql](./foundations/mechanisms/vendor-web-design/03-graphql.md)
* [04-token-refresh-design](./foundations/mechanisms/vendor-web-design/04-token-refresh-design.md)


### Foundations → Principles

High-level engineering philosophy and scalable development patterns.

Focuses on:

* system coherence
* scaling discipline
* maintainability
* defensive engineering
* production readiness
* organizational leverage


#### Consumer Facing

* [01-api-creation](./foundations/principles/consumer-facing/01-api-creation.md)
* [02-dashboard](./foundations/principles/consumer-facing/02-dashboard.md)
* [03-multi-tenant-qbo](./foundations/principles/consumer-facing/03-multi-tenant-qbo.md)


#### Dev

* [01-pipeline](./foundations/principles/dev/01-pipeline.md)
* [02-development-process](./foundations/principles/dev/02-development-process.md)
* [03-scalable-development-philosophy](./foundations/principles/dev/03-scalable-development-philosophy.md)
* [04-final-step-to-production](./foundations/principles/dev/04-final-step-to-production.md)
* [05-structure-README](./foundations/principles/dev/05-structure-README.md)
* [06-context-switching](./foundations/principles/dev/06-context-switching.md)
* [07-defensive-engineering](./foundations/principles/dev/07-defensive-engineering.md)
* [08-control-w-class](./foundations/principles/dev/08-control-w-class.md)


#### Engine

* [01-tree-walker](./foundations/principles/engine/01-tree-walker.md)


#### System General

* [00-python-dev-environment](./foundations/principles/system-general/00-python-dev-environment.md)
* [01-unit-test](./foundations/principles/system-general/01-unit-test.md)
* [02-packaging](./foundations/principles/system-general/02-packaging.md)
* [03-package-resources](./foundations/principles/system-general/03-package-resources.md)
* [04-logging](./foundations/principles/system-general/04-logging.md)
* [05-python-env](./foundations/principles/system-general/05-python-env.md)
* [06-env-example-qbo](./foundations/principles/system-general/06-env-example-qbo.md)
* [07-docstring-design](./foundations/principles/system-general/07-docstring-design.md)
* [08-product-packaging](./foundations/principles/system-general/08-product-packaging.md)
* [09-vscode-extension](./foundations/principles/system-general/09-vscode-extension.md)


### Foundations → Decisions

Architectural decisions and historical pivots.

Used to document:

* why a direction was chosen
* alternatives considered
* operational constraints
* long-term consequences

##### All Projects

* [01-input-ingestion](./foundations/decisions/all-projects/01-input-ingestion.md)

##### QBO Refactor

* [01-file-write-across-os](./foundations/decisions/qbo-refactor/01-file-write-across-os.md)
* [02-wsl-pivot-decision](./foundations/decisions/qbo-refactor/02-wsl-pivot-decision.md)

## References

`references/` contains high-speed operational lookup material.

These are:

* commands
* syntax reminders
* common operations
* quick examples
* practical workflows

### Git

* [00-setup](./references/git/00-setup.md)
* [01-common-operation](./references/git/01-common-operation.md)
* [02-branch](./references/git/02-branch.md)
* [03-stash-workflow](./references/git/03-stash-workflow.md)


### Pandas

* [00-setup](./references/pandas/00-setup.md)
* [01-io-inspect](./references/pandas/01-io-inspect.md)
* [02-query-selection](./references/pandas/02-query-selection.md)
* [03-modify-data](./references/pandas/03-modify-data.md)
* [04-advanced](./references/pandas/04-advanced.md)


### Python General

* [01-data-structures-and-utils](./references/python-general/01-data-structures-and-utils.md)
* [02-strings-and-text](./references/python-general/02-strings-and-text.md)
* [03-files-and-paths](./references/python-general/03-files-and-paths.md)
* [04-datetime-and-time](./references/python-general/04-datetime-and-time.md)
* [05-functions-and-arguments](./references/python-general/05-functions-and-arguments.md)
* [06-imports-packages-and-runtime](./references/python-general/06-imports-packages-and-runtime.md)
* [07-iterables](./references/python-general/07-iterables.md)
* [08-io-json](./references/python-general/08-io-json.md)
* [09-type-hint](./references/python-general/09-type-hint.md)
* [10-raise-errors](./references/python-general/10-raise-errors.md)
* [11-regex](./references/python-general/11-regex.md)
* [12-generator](./references/python-general/12-generator.md)


### Spark

* [00-setup](./references/spark/00-setup.md)
* [01-core-operation](./references/spark/01-core-operation.md)
* [02-column-expressions](./references/spark/02-column-expressions.md)
* [03-aggregation](./references/spark/03-aggregation.md)
* [04-string](./references/spark/04-string.md)
* [05-date](./references/spark/05-date.md)
* [06-collections](./references/spark/06-collections.md)
* [07-advanced](./references/spark/07-advanced.md)
* [08-window](./references/spark/08-window.md)
* [09-parsing](./references/spark/09-parsing.md)
* [10-UDF](./references/spark/10-UDF.md)
* [11-other-operation](./references/spark/11-other-operation.md)
* [12-pandas-to-spark](./references/spark/12-pandas-to-spark.md)


### Project Specific

* [01-structure-repo](./references/project-specific/01-structure-repo.md)
* [02-container-for-defaults](./references/project-specific/02-container-for-defaults.md)
* [03-schema-enforcement](./references/project-specific/03-schema-enforcement.md)
* [04-test-vs-function-split](./references/project-specific/04-test-vs-function-split.md)
* [05-production-example](./references/project-specific/05-production-example.md)


### Misc

* [special-symbols](./references/misc/special-symbols.md)


## Design Goals

It prioritizes:

* structure
* explicit categorization
* naming consistency
* indexed progression
* durable concepts over scattered notes


## Long-Term Vision

The long-term goal is to evolve this into:

* a reusable engineering knowledge system
* a mechanism library
* an architectural decision archive
* a defensive engineering playbook
* a scalable systems-thinking repository

