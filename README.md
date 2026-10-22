# Medical Claims Cost Trend — AtScale SML model

This is an AtScale semantic model for health-plan **medical cost trend**. It covers PMPM, utilization per 1,000, unit cost and medical loss ratio, broken out by line of business, state/market, service category, provider, diagnosis, DRG and member. Every measure drills down to the individual **claim** and **claim line**.

It targets Databricks (Unity Catalog). Model `claims_cost_trend` sits in catalog `claims_cost_trend_catalog`.

## Inputs

| File in `context/` | Description |
|---|---|
| `use_case.md` | Use case: scenario, 8 brief NLQs, engineered stories, grain, measure definitions, drill axes, limitations |
| `use_case.md` (NLQs section) | NLQs: embedded in the use case. No separate NLQ file was provided. |
| `nlq_smoke_tests.sql` | SQL reference answers for each NLQ, including the claim-level drills (used to choose measures) |
| `ddl.sql` | Databricks DDL for the 12 source tables (catalog/schema are loader placeholders in the source) |
| `erd.mmd` | Mermaid ERD of the star schema (3 facts, 9 dimensions) |
| `data_profile.yaml` | Column-level profile: row counts, distinct/null counts, roles, FKs, additivity |
| `manifest.json` | Bundle manifest: window, row counts, narratives, load defaults |
| `build.yaml` | Effective build configuration, assembled from the prompt and skill defaults |
| Target warehouse | Databricks, catalog `main`, schema `hcsc_claims_cost_trend` (stated in the prompt) |
| Existing SML | (not provided) |

## Build parameters

| Parameter | Value |
|---|---|
| `unrelated_dimension_handling` | `repeat` |
| `warehouse` | Databricks (`as_connection: Databricks`) |
| `database` / `schema` | `main` / `hcsc_claims_cost_trend` |
| `model_unique_name` | `claims_cost_trend` |
| `catalog_unique_name` | `claims_cost_trend_catalog` |
| `currency` | USD |
| `time_window` | calendar |
| `use_cases_covered` | all (medical cost trend, high-cost claimants, provider price variation, claim and claim-line drill) |
| `use_cases_excluded` | none from this model. The bundle itself excludes claims-ops SLAs, FWA, network adequacy and pharmacy benefit. |
| `semi_additive_default` | `sum`. All three facts are transactional or flow facts, so no semi-additive metrics are emitted. |

## Assumptions and decisions

- **warehouse = Databricks**: stated in the prompt. Identifiers are emitted lowercase (Rule 1).
- **database = main, schema = hcsc_claims_cost_trend**: the source DDL parameterizes the catalog and schema with loader placeholder tokens. The concrete values were taken from the prompt, which match the bundle's `load.sh` defaults (Rule 17). Override them in `connections/` if you loaded into a different catalog or schema.
- **as_connection = Databricks**: this must match the name of the Databricks data-warehouse connection in your AtScale instance. Rename it if your connection is called something else.
- **Customer name kept out of logical metadata (Rule 18)**: model, catalog, connection, dimensions and metrics use generic names. The one retained reference is the **physical schema name** `hcsc_claims_cost_trend`. It is kept so the model resolves. To remove it, reload the data into a generically named schema and repoint the connection and the two query datasets.
- **model_unique_name = claims_cost_trend**: derived from the use case title ("Medical Claims Cost Trend"), with the customer name dropped.
- **currency = USD**: stated in the use case.
- **time_window = calendar**: the use case says calendar years, with no fiscal or retail markers.
- **unrelated_dimension_handling = repeat**: skill default (Rule 3), applied to all 27 base metrics. This is what makes cross-fact ratios work. For example, Member Months (eligibility) repeats across Service Category, Provider, Diagnosis, DRG and Claim, so *Allowed PMPM by service category* divides by total member months.
- **Conformed date (Rule 21)**: all three facts join the plain `Date Dimension`: claim headers on `service_from_date_key`, claim lines on `service_date_key`, and eligibility on `month_key` (the first of the month). This gives PMPM, per-1,000 and MLR a shared time axis on an incurred (service-month) basis. Header `incurred_month_key` equals the month of `service_from_date_key`, so the incurred-month column is not modeled separately.
- **Role-play "Adjudicated {0}"**: a second date on the claim header (`adjudicated_date_key`), used for payment-lag and completion analysis. This is the only role-play.
- **Time intelligence**: `ParallelPeriod` on `[Date Dimension].[Date Hierarchy].[Year]`, with a `prior_year` calculated column and `parallel_periods` on Quarter and Month (Rule 13). Year to date uses `Aggregate(PeriodsToDate(...))`.
- **Line fact as a query dataset (Rule 15)**: the physical `fact_claim_line` has no geography, plan or DRG keys. Its dataset is a SQL view that joins the claim header, using fully-qualified table names, to add `geo_id`, `plan_id` and `drg_procedure_id`. Line measures such as J-code allowed then slice by State, Line of Business and DRG.
- **DRG dimension from a query dataset (Rule 15)**: `dim_drg` selects the `MS-DRG` rows of `dim_procedure`, so the DRG list contains no CPT, HCPCS or revenue codes. Line procedures use the separate Procedure Dimension.
- **Claim dimension sourced from the claim header fact (Rule 20 pattern)**: `fact_claim_header` backs the Claim dimension and joins it 1:1 on `claim_id`. Claim lines also join it on `claim_id`, so status, denial reason, place of service and the claim number apply to both facts.
- **Claim Line Dimension is degenerate**: `line_number` and `line_status` come from the line fact. It has `is_degenerate: true`, is listed in the model's `dimensions:` block, and has no `is_unique_key` (Rule 12 and Rule 16).
- **Provider location is snowflaked**: `dim_provider.geo_id` connects to Provider Market, which is bound to `dim_geography` inside the Provider Dimension (Rules 5 and 9). Member geography is the separate Geography Dimension, joined from the facts.
- **Line of business is snowflaked**: `dim_plan.lob_id` connects to the Line of Business level, and all facts join at the Plan level.
- **is_unique_key**: set only on single-column dimension primary keys whose profile `distinct_count` equals `row_count`: date_key, geo_id, lob_id, plan_id, service_category_id, diagnosis_id, procedure_id (the DRG subset is unique too), provider_id, member_id and claim_id. It is omitted on rollup levels and the degenerate dimension (Rule 16).
- **Calculated columns added on `fact_claim_header`**: `is_paid` and `is_denied` (claim counts and denial rate); `inpatient_days` and `inpatient_allowed_amount` (ALOS and allowed per admission, paid admissions only); `payment_lag_days`; and `insured_paid_amount`. `insured_paid_amount` is the paid amount where `lob_id <> 2`, which excludes self-funded ASO (lob_id 2 in `dim_line_of_business`), and is the MLR numerator. If LOB ids change, update this column.
- **Calculated label columns**: `dim_date` (`prior_year`, `year_name`, `month_label`, `day_label`, `month_completeness`), `dim_service_category` (BH and specialty-drug flag labels) and `dim_diagnosis` (`diagnosis_label`, `chronic_flag`). These are display strings for booleans and codes.
- **Ratios use `Divide()`**: this returns blank on a zero denominator, so MLR is blank for ASO-only slices, where premium is 0.
- **Not modeled**: the attributed PCP on `dim_member` (`pcp_provider_id`) is not joined. The use case has no PCP-attribution NLQ, and an embedded member-to-provider path would conflict with the billing-provider join. Header `received_date_key` and `service_thru_date_key` are not role-played; `payment_lag_days` covers lag analysis.
- **Ranking (top-N claimants)**: there is no TopCount. Answer "top high-cost claimants" by sorting Member by Paid Amount in the BI or NLQ layer. MDX ranking functions are not supported (INDEX.md).

## Generation summary

| Object | Count |
|---|---|
| Datasets | 13: 11 physical tables, plus 2 query datasets (`dim_drg`, and `fact_claim_line` enriched with header keys) |
| Dimensions | 11: Date (time), Geography, Line of Business, Member, Provider, Service Category, Diagnosis, Procedure, DRG, Claim, and Claim Line (degenerate) |
| Base metrics | 27: membership 3, financial 3, utilization 8, cost 5, claim lines 8 |
| Calculated metrics | 31: PMPM, MLR, per-1,000, unit cost, ratios, and prior-year / YTD / YoY time intelligence |
| Model relationships | 24: eligibility 4, claim header 10, claim line 10 |

- **Role-play prefixes**: `Adjudicated {0}` on Date Dimension (claim header only).
- **Snowflake bridges**: `dim_plan` connects to Line of Business; `dim_provider` connects to Provider Market.
- **NLQ coverage**:
  1. PMPM and MLR by LOB and year
  2. TX MA PMPM, admits per 1,000 by Acquisition Cohort, then DRG, then Claim
  3. Allowed PMPM by Service Category, then Procedure (J-code), then Claim
  4. Paid Claims per 1,000 with the Behavioral Health Flag, by State
  5. Paid Amount by Member, then Claim
  6. Allowed per Admission by Health System and Provider, filtered to DRG 470
  7. ER Visits by Month, Diagnosis Category and Diagnosis, then Claim
  8. Claim, Line Number and Procedure, with line billed, allowed, deductible, copay, coinsurance and paid
- **Caveats**:
  - 2026 is partial, and Aug–Sep 2026 are incomplete because of claims lag. Use the `Month Completeness` attribute.
  - Cost per admission and ALOS are meaningful when filtered to inpatient service categories or DRGs.
  - The Claim dimension has about 522K members. Filter it first, for example by member, DRG or date, before drilling to claims.

## Reproducing this build

`context/` holds verbatim copies of every input: the use case, DDL, ERD, data profile, NLQ smoke tests, bundle manifest and the effective `build.yaml`. Re-running the AtScale SML model generator skill with these inputs and build parameters should produce an equivalent model. The generator script (`emit_sml.py`) is deterministic.
