# DevOps Essential Index

> Purpose: fast recall under pressure.  
> This is not a table of contents — it is a **decision map**.

## 0. If something feels fragile or risky

- **Input & ingestion safety**
  - [User Input Ingestion Methods](https://github.com/ZheRao/NoteSystem/blob/main/devops/foundations/decisions/all-projects/01-input-ingestion.md)

- **File write safety (atomic, cross-OS)**
  - [File Writes Across OS](https://github.com/ZheRao/NoteSystem/blob/main/devops/foundations/decisions/qbo-refactor/01-file-write-across-os.md)

- **WSL pivot rationale (why Linux-first)**
  - [Linux Pivot](https://github.com/ZheRao/NoteSystem/blob/main/devops/foundations/decisions/qbo-refactor/02-wsl-pivot-decision.md)


## 1. Repo & system structure 

- **How repos should be structured**
  - [Repo and Import](https://github.com/ZheRao/NoteSystem/blob/main/devops/references/project-specific/01-structure-repo.md)
  - [Package Data instead of Code](https://github.com/ZheRao/NoteSystem/blob/main/devops/foundations/principles/system-general/03-package-resources.md)

- **Make contracts specific and scalable**
  - [Container Objects for Defaults & Config](https://github.com/ZheRao/NoteSystem/blob/main/devops/references/project-specific/02-container-for-defaults.md)
  - [Python Type Hint](https://github.com/ZheRao/NoteSystem/blob/main/devops/references/python-general/09-type-hint.md)


## 2. I/O, files, paths, and JSON

- **Files & paths (Pathlib, OS behavior)**
  - [File System Basics](https://github.com/ZheRao/NoteSystem/blob/main/devops/references/python-general/03-files-and-paths.md)

- **JSON I/O (text vs bytes, orjson, ujson)**
  - [I/O for JSON](https://github.com/ZheRao/NoteSystem/blob/main/devops/references/python-general/08-io-json.md)


## 3. Environment, runtime, and dependencies

- **Python environments (venv, pip, runtime)**
  - [Venv Setup](https://github.com/ZheRao/NoteSystem/blob/main/devops/foundations/principles/system-general/05-python-env.md)


## 4. Develop scalable system

- **Why unit tests exist (when to add / skip)**
  - [TDD](https://github.com/ZheRao/NoteSystem/blob/main/devops/foundations/principles/system-general/01-unit-test.md)

- **Logging (what to log, where, why)**
  - [Logging](https://github.com/ZheRao/NoteSystem/blob/main/devops/foundations/principles/system-general/04-logging.md)

## 5. Spark

- **Spark execution & orchestration**
  - [Core Orchestration](https://github.com/ZheRao/NoteSystem/blob/main/devops/foundations/mechanisms/spark/01-core-orchestration.md)
  - [Execution Timeline](https://github.com/ZheRao/NoteSystem/blob/main/devops/foundations/mechanisms/spark/02-spark-exe-timeline.md)

- **Data behavior pitfalls**
  - [Repartition](https://github.com/ZheRao/NoteSystem/blob/main/devops/foundations/mechanisms/spark/03-repartition-behavior.md)
  - [Schema](https://github.com/ZheRao/NoteSystem/blob/main/devops/foundations/mechanisms/spark/04-schema-handling.md)

- **Spark API reference**
  - [Core Operations](https://github.com/ZheRao/NoteSystem/blob/main/devops/references/spark/01-core-operation.md)
  - [Higher-order Functions](https://github.com/ZheRao/NoteSystem/blob/main/devops/references/spark/07-advanced.md)

## 7. Git

- **Setup & sanity**
  - [Setup](https://github.com/ZheRao/NoteSystem/blob/main/devops/references/git/00-setup.md)

- **Daily operations**
  - [Daily Ops](https://github.com/ZheRao/NoteSystem/blob/main/devops/references/git/01-common-operation.md)

- **Branches & stashing**
  - [Branch](https://github.com/ZheRao/NoteSystem/blob/main/devops/references/git/02-branch.md)
  - [Stash](https://github.com/ZheRao/NoteSystem/blob/main/devops/references/git/03-stash-workflow.md)

---

## 8. Pandas

- **I/O & inspection**
  - [Creation](https://github.com/ZheRao/NoteSystem/blob/main/devops/references/pandas/01-io-inspect.md)

- **Querying & transformation**
  - [Observe](https://github.com/ZheRao/NoteSystem/blob/main/devops/references/pandas/02-query-selection.md)
  - [Modify](https://github.com/ZheRao/NoteSystem/blob/main/devops/references/pandas/03-modify-data.md)

- **Advanced patterns**
  - [Advanced Mechanisms](https://github.com/ZheRao/NoteSystem/blob/main/devops/references/pandas/04-advanced.md)
  - [Regex](https://github.com/ZheRao/NoteSystem/blob/main/devops/references/pandas/05-regex.md)


## Rule of use
- If you *searched twice*, add a link here.
- If something broke once, document **why**, not just how.
- This file must stay **short and decisive**.
