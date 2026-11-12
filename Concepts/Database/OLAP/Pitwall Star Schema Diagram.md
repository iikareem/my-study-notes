---
aliases:
  - Pitwall Star Schema
  - F1 Star Schema Diagram
tags:
  - olap
  - pitwall
  - schema
---

# Pitwall — Kimball Star Schema (visual)

> Open this note in Obsidian — Mermaid renders below.  
> Static image: ![[pitwall-star-schema.png]]

**Design name:** Kimball dimensional modeling → **star schema**  
**Rule:** facts join dims directly. Facts do **not** depend on each other. Dims are **shared**.

---

## ERD style (entities + keys + relationships)

```mermaid
erDiagram
  dim_drivers {
    varchar driver_id PK
    varchar driver_code
    varchar given_name
    varchar family_name
    varchar nationality
    date date_of_birth
  }

  dim_constructors {
    varchar constructor_id PK
    varchar constructor_name
    varchar nationality
  }

  dim_circuits {
    varchar circuit_id PK
    varchar circuit_name
    varchar locality
    varchar country
    double lat
    double long
  }

  dim_races {
    int season PK
    int round PK
    varchar race_name
    date race_date
    varchar circuit_id FK
  }

  fact_results {
    int season FK
    int round FK
    varchar driver_id FK
    varchar constructor_id FK
    varchar circuit_id FK
    int grid
    int finish_position
    double points
    varchar status
    int laps_completed
    int race_time_ms
    int fastest_lap_ms
  }

  fact_qualifying {
    int season FK
    int round FK
    varchar driver_id FK
    varchar constructor_id FK
    varchar circuit_id FK
    int q1_ms
    int q2_ms
    int q3_ms
    int qualify_position
  }

  fact_laps {
    int season FK
    int round FK
    varchar driver_id FK
    varchar circuit_id FK
    int lap
    int position
    int time_ms
  }

  fact_pit_stops {
    int season FK
    int round FK
    varchar driver_id FK
    int stop
    int lap
    int duration_ms
  }

  dim_drivers ||--o{ fact_results : "driver_id"
  dim_constructors ||--o{ fact_results : "constructor_id"
  dim_circuits ||--o{ fact_results : "circuit_id"
  dim_races ||--o{ fact_results : "season,round"

  dim_drivers ||--o{ fact_qualifying : "driver_id"
  dim_constructors ||--o{ fact_qualifying : "constructor_id"
  dim_circuits ||--o{ fact_qualifying : "circuit_id"
  dim_races ||--o{ fact_qualifying : "season,round"

  dim_drivers ||--o{ fact_laps : "driver_id"
  dim_circuits ||--o{ fact_laps : "circuit_id"
  dim_races ||--o{ fact_laps : "season,round"

  dim_drivers ||--o{ fact_pit_stops : "driver_id"
  dim_races ||--o{ fact_pit_stops : "season,round"
```

**How to read the crow’s foot:** `||--o{` means **one** dim row → **many** fact rows (classic star).

---

## Diagram

```mermaid
flowchart TB
  subgraph dims ["DIMENSIONS — shared lookups"]
    direction LR
    dim_drivers["dim_drivers\nwho"]
    dim_constructors["dim_constructors\nteam"]
    dim_circuits["dim_circuits\ntrack"]
    dim_races["dim_races\nweekend"]
  end

  subgraph facts ["FACTS — independent grains"]
    direction LR
    fact_results["fact_results\n1 row = driver × race"]
    fact_qualifying["fact_qualifying\n1 row = driver quali"]
    fact_laps["fact_laps\n1 row = driver lap"]
    fact_pit_stops["fact_pit_stops\n1 row = one stop"]
  end

  fact_results -->|driver_id| dim_drivers
  fact_results -->|constructor_id| dim_constructors
  fact_results -->|circuit_id| dim_circuits
  fact_results -->|"season, round"| dim_races

  fact_qualifying -->|driver_id| dim_drivers
  fact_qualifying -->|constructor_id| dim_constructors
  fact_qualifying -->|circuit_id| dim_circuits
  fact_qualifying -->|"season, round"| dim_races

  fact_laps -->|driver_id| dim_drivers
  fact_laps -->|circuit_id| dim_circuits
  fact_laps -->|"season, round"| dim_races

  fact_pit_stops -->|driver_id| dim_drivers
  fact_pit_stops -->|"season, round"| dim_races
```

---

## Same idea as a star (one fact in the middle)

Example for **results only** — the other facts look the same, just different measures/grain:

```mermaid
flowchart TB
  DD[dim_drivers]
  DC[dim_constructors]
  DCI[dim_circuits]
  DR[dim_races]

  FR["fact_results\nkeys + points, grid, status, …"]

  FR --> DD
  FR --> DC
  FR --> DCI
  FR --> DR
```

---

## Remember

| Piece | Role |
| --- | --- |
| **Fact** | What happened (measures / events) |
| **Dim** | Who / what / where labels |
| **Many facts** | Different grains — independent files |
| **Shared dims** | One `dim_drivers` used by all facts |

**Parent:** [[00 OLAP MOC]] · [[OLAP Starter Path — Think OLAP First]]
