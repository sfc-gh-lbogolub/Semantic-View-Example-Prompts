# Example Prompts: Building Semantic Views with Cortex Code

A library of prompts for building, validating and refining Snowflake semantic views with Cortex Code (CoCo). Each prompt has:
- **When to use it**
- **The prompt**, shown with a sales-and-fulfillment example; swap in your own tables and terms
- **Why it works**: the best practice it encodes
- **Template**: a fill-in version for your own data

The examples use one semantic view over two fact tables (orders and shipments) that share product, customer and calendar dimensions. The same patterns work for a single-fact view.

---

## 1. Discover before you model

### 1.1 Profile the data
**When to use it:** at the start, before CoCo writes any DDL.

> Explore the tables in ANALYTICS_DB.SALES. For each one, tell me its grain, primary key and foreign keys. Identify which tables are facts and which are dimensions, and how each dimension joins to the facts. If there's more than one fact table, also tell me which dimensions they share and which apply to only one fact. Profile key columns: date ranges, units of measure, currencies, and distinct values of categorical columns. Don't build anything yet.

**Why it works**
- **Grain, keys and units cause most accuracy problems:** fan-out, wrong join paths, summing values in different units or currencies.
- **Asking about shared dimensions only "if there's more than one fact"** makes the same prompt work for single-fact and multi-fact views, and tells you early whether you need the multi-fact design in 2.2.
- **"Don't build anything yet"** keeps CoCo in research mode so you can check what it found first.

**Template**
> Explore the tables in `<DB>.<SCHEMA>`. For each table, give its grain, primary key and foreign keys. Identify the facts and dimensions and how they join. If there are multiple facts, list the dimensions they share and those used by only one fact. Profile `<key columns>`: date ranges, units, distinct values. Don't build anything yet.

### 1.2 Pull business context from existing queries
**When to use it:** when the team already has dashboards or SQL that define the KPIs.

> Look at how these tables are queried today (query history, existing views and any SQL I paste below). List the metrics people calculate, the exact formulas they use, and common filters. Flag any metric that's calculated more than one way.

**Why it works:** real usage tells you which metrics matter and where definitions conflict. Settle the conflicts **before** the definition goes into the semantic view.

**Template**
> Review how `<tables>` are queried today (query history, views `<names>`, and this SQL: `<paste>`). List the metrics, their formulas and common filters, and flag conflicting definitions.

---

## 2. Design the model (use plan mode)

### 2.1 Single-fact semantic view
**When to use it:** for one business process with one fact table.

> In plan mode, design a semantic view `ANALYTICS_DB.SALES.SV_SALES` over FACT_ORDERS with DIM_PRODUCT, DIM_CUSTOMER and DIM_SALES_REP. Requirements:
> - A primary key and a relationship for every join.
> - Metrics: total_revenue, total_margin, margin_pct (**as a ratio of sums, not an average**), order_count, avg_order_value.
> - A description on every metric and dimension, with the synonyms our team uses (sales, bookings, gross profit, account, rep).
> - Sample values for low-cardinality dimensions (region, product_category, channel).
> - A rule that quantity and unit price are only meaningful within a single product.
> - 4 verified queries with **absolute dates** (e.g. Q2 2026), not "last quarter".

**Why it works**
- **Plan mode** lets you review the design before any DDL runs.
- **"Ratio of sums"** prevents the most common metric error, averaging percentages.
- **Descriptions, synonyms and sample values** are how Cortex Analyst maps business language to columns.
- **Absolute dates** keep verified queries correct over time, so they can also serve as evaluation ground truth.

**Template**
> In plan mode, design semantic view `<DB.SCHEMA.NAME>` over `<fact>` with `<dimensions>`. Requirements: primary keys and relationships for every join; metrics `<list, with formulas for ratios>`; descriptions and real business synonyms (`<terms>`) on every element; sample values for `<low-cardinality dims>`; rules: `<unit / additivity rules>`; `<N>` verified queries with absolute dates.

### 2.2 Multi-fact semantic view with shared dimensions
**When to use it:** for two or more facts that people analyze together.

> In plan mode, design ONE semantic view `SV_SALES_FULFILLMENT` over FACT_ORDERS and FACT_SHIPMENTS. Requirements:
> - **Star schema with two facts:** both facts relate to the shared DIM_PRODUCT, DIM_CUSTOMER and DIM_DATE (orders on order_date, shipments on ship_date). DIM_SALES_REP relates only to orders and DIM_CARRIER only to shipments.
> - **No relationship between the two fact tables.** Keep shipment order_id as a plain attribute so there's one join path to each shared dimension.
> - **Each metric on its own fact:** orders get revenue, margin, margin_pct and order_count. Shipments get shipping cost, shipping cost per unit, shipment count, transit days, on-time % and damage %.
> - **Derived cross-fact metrics:** shipping_pct_of_revenue, margin_after_shipping, margin_after_shipping_pct, shipping_impact_pts.
> - **Custom instructions:** cross-fact questions may only group or filter by shared dimensions (use the calendar for dates); shipping cost can't be broken down by sales rep; per-unit metrics only within one product.
> - Descriptions, synonyms, sample values, and 5 verified queries with absolute dates, including at least 2 cross-fact questions.

**Why it works**
- **Shared (conformed) dimensions are the only safe link between facts.** A fact-to-fact join creates a second path to each shared dimension, which risks double-counting (fan-out).
- **A shared calendar is required.** Each fact has its own date column, so without a common date dimension you can't filter both facts to the same period.
- **Each fact aggregates at its own grain.** Derived metrics then combine the results, which is what keeps cross-fact ratios correct.
- **Custom instructions** spell out which combinations are valid.

**Template**
> In plan mode, design ONE semantic view `<NAME>` over `<fact A>` and `<fact B>`. Both relate to shared `<dims>` and a shared calendar (`<fact A date>`, `<fact B date>`). `<dim X>` relates only to `<fact A>`; `<dim Y>` only to `<fact B>`. No relationship between the facts. Each metric on its own fact: `<lists>`. Derived cross-fact metrics: `<formulas>`. Custom instructions: `<valid combinations and rules>`. Descriptions, synonyms, sample values, and `<N>` verified queries with absolute dates.

### 2.3 Add a shared calendar if one doesn't exist
**When to use it:** when the facts use different date columns and there's no date dimension.

> Create a calendar table `<DB.SCHEMA>.DIM_DATE` with one row per day from `<start date>` to `<end date>`, with columns date_day, month_start, quarter_start, year, quarter_label (e.g. "Q2 2026") and month_label. If we use a fiscal calendar, add fiscal_year and fiscal_quarter too. Then relate both facts to it in the semantic view.

**Why it works:** this is what makes "Q2 2026" mean the same thing for every fact, and it prevents *"entities are not related"* errors on cross-fact date filters.

---

## 3. Validate before anyone uses it

### 3.1 Fan-out check
**When to use it:** right after creating the view, and after every relationship change.

> Validate `SV_SALES_FULFILLMENT`: compare total revenue, total margin, total shipping cost and shipment count from the semantic view with SUM/COUNT directly on FACT_ORDERS and FACT_SHIPMENTS. They should match when rounded to the dollar. If they don't, explain which relationship is causing it.

**Why it works:** this is the cheapest correctness test there is. A mismatch almost always means a bad join path. "Rounded to the dollar" allows for tiny floating-point differences.

**Template**
> Compare `<metrics>` from `<semantic view>` with direct SUM/COUNT on `<base tables>`. They should match (rounded). If not, identify the relationship causing the difference.

### 3.2 Test real business questions
**When to use it:** after the fan-out check passes.

> Test the semantic view with these questions and show the SQL it generates for each: (1) revenue and margin by product category in Q2 2026; (2) margin % by region by quarter; (3) top 5 West region customers by revenue in Q2 2026.

**Why it works:** checking the generated SQL, not just the answer, shows join paths, filters and aggregation problems early.

**Template**
> Test the semantic view with: `<3–5 real questions with absolute dates>`. Show the generated SQL for each.

### 3.3 Test the cross-fact payoff
**When to use it:** for multi-fact views.

> Using the semantic view, show revenue, margin %, shipping cost, shipping % of revenue, margin after shipping % and shipping impact for the Electronics category in the West region in Q2 2026.

**Why it works:** it confirms the derived metrics and the shared calendar work together in one query.

### 3.4 Test the guardrails
**When to use it:** for multi-fact views, to confirm that invalid combinations fail.

> Now show total shipping cost by sales rep. Tell me what happens and why.

**Why it works:** a well-modeled view **refuses** combinations that aren't related (Snowflake returns *"entities are not related"*) rather than returning a number that looks plausible but is wrong. Check that the custom instructions tell the agent to explain this to users.

**Template**
> Show `<metric from fact B>` by `<dimension only related to fact A>`. Tell me what happens and why.

### 3.5 Test under a restricted role
**When to use it:** when the underlying tables have row access or masking policies.

> Run the cross-fact query from 3.3 as role `<RESTRICTED_ROLE>`. Check that every fact table is filtered consistently, and flag any metric that combines filtered and unfiltered data.

**Why it works:** in a multi-fact view, a policy on only one fact makes cross-fact ratios wrong for restricted users, with no error. Every fact needs aligned policies.

---

## 4. Refine the view (iterate on real questions)

Each of these can also be done by hand in **Semantic Studio** (AI & ML » Cortex Analyst, then select the view). CoCo shows each change as a diff you approve.

### 4.1 Teach business jargon (descriptions)
**When to use it:** when users use terms the model doesn't recognize.

> Users say "laptops", "notebooks" and "portables" for the Computers category. Update the product_category description to say so. Confirm total_margin has the synonyms gross profit, GP and contribution.

**Why it works**
- **Synonyms rename columns and metrics.** Alternative names for **values** (notebooks for Computers) belong in descriptions or custom instructions.
- **Write synonyms from the business's real vocabulary.** Snowflake recommends against auto-generated synonyms.

**Template**
> Users call `<value>` "`<term 1>`", "`<term 2>`". Add that to the `<dimension>` description. Make sure `<metric>` has synonyms `<list>`.

### 4.2 Encode a business definition (named filter)
**When to use it:** for a recurring definition everyone uses.

> Add a named filter `key_accounts` on the customer table: customer_tier IN ('Enterprise', 'Strategic'), with the description "Key accounts: Enterprise and Strategic tier customers" and synonyms key accounts, top accounts and strategic customers.

**Why it works:** the definition lives in one place. Because customer is a shared dimension, the filter applies to every fact and to the cross-fact metrics.

**Template**
> Add a named filter `<name>` on `<table>`: `<boolean expression>`, with description "`<meaning>`" and synonyms `<terms>`.

### 4.3 Add a governed KPI (metric or derived metric)
**When to use it:** when users keep asking for a KPI the view doesn't define.

> Add a metric total_cost = SUM(cost) on orders, then a derived metric shipping_share_of_cost = total_shipping_cost / total_cost, described as "Shipping as a fraction of total order cost." Test it for the West region in Q2 2026.

**Why it works:** a KPI defined in the semantic view is calculated the same way by every agent, app and BI tool.

**Template**
> Add metric `<name>` = `<aggregate expression>` on `<table>` (and derived metric `<name>` = `<formula>`), described as "`<business meaning>`". Test it with `<question>`.

### 4.4 Add a rule (custom instructions)
**When to use it:** when answers are technically valid but wrong for the business.

> Add a custom instruction: "Never average shipping cost per unit across products; break down by product and unit, or report total shipping cost instead."

**Why it works:** custom instructions hold the rules a data owner would explain to a new analyst (units, exclusions, defaults). Keep them short. Display formatting belongs in the agent's response instructions, not here.

**Template**
> Add a custom instruction: "`<rule, phrased as never/always + what to do instead>`".

### 4.5 Turn good answers into verified queries
**When to use it:** after a question returns a correct, important answer.

> That answer is correct. Add it to the semantic view as a verified query with the question "How much did shipping costs reduce West region Electronics margins in Q2 2026?", using absolute dates, and mark it as an onboarding question.

**Why it works:** verified queries steer SQL generation for similar questions (known questions reuse the exact governed SQL), become evaluation ground truth, and onboarding questions show up as starters for users.

**Template**
> Add that as a verified query: question "`<question>`", absolute dates, onboarding question `<true/false>`.

### 4.6 Fix verified queries that use relative dates
**When to use it:** when reviewing a view someone else built.

> Review all verified queries in the semantic view. Rewrite any that use CURRENT_DATE, "last quarter" or "this year" to use fixed date ranges, and update the question text to match.

**Why it works:** relative dates make verified queries and evaluations drift as time passes.

### 4.7 Debug a wrong answer
**When to use it:** when a user reports an incorrect result.

> This question returned the wrong number: "`<question>`". The correct answer is `<value>`. Show me the SQL that was generated, explain what went wrong (join path, filter, metric definition or missing context), and propose the smallest fix to the semantic view.

**Why it works:** it fixes the root cause in the semantic layer rather than patching the prompt, so every consumer benefits.

---

## 5. Evaluate and keep it healthy

### 5.1 Run an evaluation
**When to use it:** before going live, and after significant changes.

> Set up a Cortex Analyst evaluation for the semantic view using its verified queries, pinned to the latest SQL correctness metric version. Run it and summarize accuracy, failures and what to change.

**Why it works:** it gives you a measured baseline, and it catches regressions when the model changes. Pin the metric version so runs stay comparable.

### 5.2 Review a semantic view against best practices
**When to use it:** for periodic reviews, or when inheriting a view.

> Audit the semantic view: check for missing descriptions, ratio metrics defined as averages, fact-to-fact relationships, multiple join paths, relative dates in verified queries, missing sample values, and dimensions not shared across facts. List the issues by severity with a proposed fix for each.

---

## Prompting tips
- **Name objects fully** (`DB.SCHEMA.OBJECT`) and spell out the formula for any ratio metric.
- **Say what not to do:** "no fact-to-fact joins", "don't build yet", "absolute dates only".
- **Use plan mode for design,** then iterate one change at a time, each with a test question.
- **Ask CoCo to show its SQL and explain the result,** not just give the answer.
- **Validate after every structural change** with the fan-out check (3.1).
- **Keep the agent's instructions short when the semantic view is rich.** Logic in the view is reusable; logic in a prompt isn't.

---

## Next steps: optimizing your semantic view

Once the view passes validation (Section 3), use this loop to keep improving accuracy and performance.

1. **Set a baseline.** Run an evaluation (5.1) on your verified queries and record the accuracy score before making changes.
   > Run a Cortex Analyst evaluation on `<semantic view>` and save the results as the baseline.

2. **Collect real questions.** Gather 20–50 questions from the people who will actually use the view, such as from existing dashboards, report requests and support tickets. Add the most important correct answers as verified queries (4.5).
   > Here are questions from our users: `<paste list>`. Run each against `<semantic view>`, show the SQL, and flag any that look wrong or ambiguous.

3. **Mine usage after launch.** Review what users ask in production and where answers fail.
   > Review recent Cortex Analyst and agent requests against `<semantic view>`. Group the questions by theme, list the ones that failed or got low feedback, and suggest new verified queries, synonyms or metrics to cover the gaps.

4. **Fix the root cause, one change at a time.** For each failure, use 4.1–4.4 and 4.7 to make the smallest fix in the semantic view (description, synonym, filter, metric or custom instruction), then re-test that question.

5. **Re-run the evaluation after each batch of changes.** Compare against the baseline. Keep changes that improve accuracy; roll back those that cause regressions.

6. **Optimize query performance.** If answers are correct but slow, ask CoCo to find bottlenecks.
   > Analyze the queries generated by `<semantic view>` over the last 30 days. Identify the slowest query patterns and recommend options such as materializations, clustering keys, or warehouse sizing.

7. **Audit regularly.** Run the best-practice audit (5.2) monthly or whenever the underlying tables change, and update sample values, date ranges and verified queries as the business evolves.

8. **Expand deliberately.** Add new facts or dimensions only when users need them, following the multi-fact rules in 2.2 (shared dimensions, no fact-to-fact joins), and repeat validation (Section 3) after each addition.
