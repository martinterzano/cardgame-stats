# Truco Profesional

> Game session tracker and analytics dashboard for a group of regular Truco players. Replaced a manual Excel workflow with a full-stack web app backed by PostgreSQL, with SQL views that produce analytics at query time. Live at [truco.martinterzano.com](https://truco.martinterzano.com).

**Stack:** Node.js · Express · Prisma · PostgreSQL · Next.js 14 · Recharts · Railway · Vercel · Cloudflare · **Status:** in production

---

## Contents

1. [What it is](#1-what-it-is)
2. [The analytics layer](#2-the-analytics-layer)
3. [Design decisions](#3-design-decisions)
4. [Architecture](#4-architecture)
5. [AI-native development model](#5-ai-native-development-model)
6. [What this project demonstrates](#6-what-this-project-demonstrates)

---

## 1. What it is

A group of five people plays Truco (an Argentine card game) every two weeks. For two years the results went into a Google Sheet and then a Looker Studio dashboard. The manual workflow had three problems: transcription errors from paper to spreadsheet, the limitation of spreadsheets as query engines for multi-year historical analysis, and the constraint of Looker Studio when the visualization needs change.

The app replaces all three parts of that workflow.

**Loading.** A form for manual entry of a game session (a list of individual hands: who played, which pair won, how much was wagered). An Excel upload path with a downloadable template and a preview before confirmation. An admin panel for corrections and verification.

**Storage.** PostgreSQL with a normalized relational model and three SQL views that compute all analytics at query time.

**Visualization.** Six charts on a public dashboard with year and all-time filters. No login required.

## 2. The analytics layer

The relational model stores two tables: one row per game session and one row per hand played within that session. Players are stored as five boolean columns on the hands table, which is the correct model for a fixed-player game but creates a challenge for aggregation: to compute per-player stats, each row needs to expand to player-level records before grouping.

Three SQL views handle this.

**`partidos_expandidos` (expanded hands)** uses a `LATERAL JOIN` to unnest the five player columns into one row per hand per player. This is essentially `UNNEST` with a key lookup, the standard SQL pattern for going from a column-per-entity structure to a long-format table without denormalization. The view adds a computed `ganador` flag per player per hand based on who was on the winning pair.

```sql
-- Shape of the transformation (condensed, not the full view)
SELECT h.id, j.name AS jugador, h.monto,
       CASE WHEN h.pareja_ganadora LIKE '%' || j.initial || '%' THEN 1 ELSE 0 END AS ganador
FROM hands h
CROSS JOIN LATERAL (VALUES
  ('Player1', 'P1'), ('Player2', 'P2'), ('Player3', 'P3'),
  ('Player4', 'P4'), ('Player5', 'P5')
) AS j(name, initial)
WHERE /* player is in this hand */
```

**`stats_jugadores` (player stats)** aggregates `partidos_expandidos` by player and year: hands won and lost, total winnings, average bet per hand, number of sessions played. A `WHERE año = $1 OR $1 = 'all'` clause makes the year filter a query parameter rather than a separate view per year.

**`stats_parejas` (pair stats)** aggregates by the canonical pair key (see §3) and year: win rate, total money won, average per hand.

All six dashboard charts read from these three views. Adding a new chart is a matter of adding a query against an existing view, not adding a new data pipeline step.

## 3. Design decisions

**Pair normalization on INSERT.** A pair `"ND"` and a pair `"DN"` are the same pair. Two options: normalize when writing to the database, or normalize in every query that groups by pair. Normalizing on INSERT (`pair.split('').sort().join('')`) means every query can group by the stored value directly. No computed column, no function call in the GROUP BY, no risk of inconsistency between applications that read from the table.

**Regular view over materialized view.** With a maximum of around 1,500 hands accumulated over five years, the performance difference between a regular view and a materialized view on this data is measured in milliseconds, not seconds. A regular view is the right default. The migration path to a materialized view (if the dashboard ever exceeds 500ms) is documented in the codebase as an explicit future option, not a forgotten possibility.

**Railway over Supabase.** An intentional learning decision: a real Express backend gives full control over HTTP behavior, middleware, and query execution, which Supabase's managed API abstracts away. For a personal project with no time pressure, the cost of building more is worth the control gained. The pattern (own REST API with Prisma as the ORM layer) is directly transferable to professional back-end work.

**Database-level constraints.** Business rules that can be expressed as invariants live in the database, not in the application layer.

```sql
CHECK ((p1::int + p2::int + p3::int + p4::int + p5::int) = 4)
CHECK (pareja_ganadora ~ '^[CDLMN]{2}$')
CHECK (monto > 0)
```

Four players per hand, valid pair codes, positive wager. These cannot be violated even by a buggy endpoint or a direct database operation.

## 4. Architecture

```
Client (browser, no login required)
      ↑
Next.js 14 frontend (Vercel)  ←→  Node.js + Express API (Railway)
                                          ↓ Prisma ORM
                                     PostgreSQL (Railway)
                                          ↓ SQL views
                                     Analytics at query time
```

**API endpoints (summary):**

```
GET/POST/PUT/DELETE  /api/jornadas         → session CRUD
GET/POST/PUT/DELETE  /api/partidos         → hand CRUD
POST                 /api/jornadas/upload  → Excel ingest
GET                  /api/stats/jugadores  → player stats (year param)
GET                  /api/stats/parejas    → pair stats (year param)
GET                  /api/stats/rachas     → last 10 sessions per player
GET                  /api/stats/acumulado  → time-series line by player
```

## 5. AI-native development model

The backend API, SQL views, and frontend were built with **Claude Code as a technical co-worker**.

**What I designed:**

- Problem framing and solution architecture (what tables, what views, what the dashboard should show and why)
- The relational model, including the normalization decision for pairs and the database constraints
- The SQL view design: the LATERAL JOIN pattern, the year-filter approach, the stats structure
- Every chart choice: what question each chart answers, what the filter options are, what the axes mean
- Infrastructure setup end-to-end (Railway, Vercel, Cloudflare, custom domain)
- Validation of every output against two years of historical Excel data

**What Claude Code executed:**

- Express API, Prisma schema, all endpoints and validation (Zod)
- SQL views (written from my specification of what each view needs to compute)
- Next.js frontend, Recharts charts, shadcn/ui components

The design is where domain knowledge lives. Claude Code handles execution speed. The result is a production system built to a quality level that would otherwise require a dedicated full-stack engineer.

## 6. What this project demonstrates

- **SQL analytics design.** The LATERAL JOIN pattern to go from column-per-entity to long format, year-filtered views as a query pattern, database constraints as a data quality layer. These are data engineering patterns applied to a personal project. The same patterns appear in professional analytics work at larger scale.
- **Design tradeoff documentation.** The normalized view vs. materialized view decision, the Railway vs. Supabase choice, the pair normalization strategy are all in the codebase with their reasoning. The documentation is part of the decision, not a follow-up task.
- **Full-stack delivery from first principles.** Custom REST API, ORM with migrations, public Next.js frontend, CDN and custom domain. End-to-end ownership without a managed platform absorbing the decisions.
- **AI-native delivery.** Designing and validating a full-stack system as a solo practitioner, using Claude Code to execute at speed while maintaining design ownership, is a concrete capability in the current development environment.

---

Martín Terzano · [helliumlab.com](https://helliumlab.com) · [LinkedIn](https://www.linkedin.com/in/martinterzano) · martin@helliumlab.com
