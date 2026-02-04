# Plus Activations BOB Attribution Dashboard

A real-time dashboard for analyzing Shopify Plus activations and their Book of Business (BOB) territory attribution at activation time.

![Dashboard Preview](https://img.shields.io/badge/Quick%20Site-Deployed-00f5d4?style=for-the-badge)
![BigQuery](https://img.shields.io/badge/BigQuery-Powered-4285F4?style=for-the-badge&logo=google-cloud)

## 🎯 Purpose

This dashboard answers key questions about Plus activations:

1. **How did merchants get to Plus?** (Path Category)
   - **Self-serve Trial** - Started a Plus trial before activating
   - **Self-serve Admin** - Self-serve upgrade via admin (no trial)
   - **Non Self-Serve** - Not trial, not in admin funnel → sales involvement

2. **Was sales already working this merchant?** (BOB Attribution)
   - Checks if the org was in a sales territory *before* they activated Plus
   - Uses the most recent territory snapshot prior to activation date

3. **Regional breakdown** - AMER, EMEA, APAC distribution

## 📊 Metrics

| Metric | Description |
|--------|-------------|
| `total_activations` | Count of distinct orgs that activated Plus |
| `in_bob_count` | Count of activations that were in a sales territory |
| `in_bob_rate_pct` | % of activations in BOB (in_bob_count / total × 100) |

## 🏗️ Data Sources

| Table | Purpose |
|-------|---------|
| `shopify-dw.mart_growth.admin_plus_metrics` | Base Plus activation data |
| `shopify-dw.mart_growth.admin_plus_activation_funnel` | Identify self-serve (Admin Channel) path |
| `shopify-dw.accounts_and_administration.organization_contract_pricing_info_current` | Contract type info |
| `sdp-stg-commercial.lucasmiller_scratch.account_territory_daily_snapshot` | Territory snapshots for BOB attribution |
| `shopify-dw.accounts_and_administration.business_platform_contracts` | Contract data |
| `shopify-dw.sales.sales_reporting_regions` | Regional mapping (country_code → region) |
| `shopify-dw.sales.sales_leads` | Plus leads data for self-serve routing analysis |

## 🚀 Deployment

### Quick Site (Recommended)

```bash
# Deploy to Quick Site
quick deploy . plus-bob-dashboard
```

Visit: `https://plus-bob-dashboard.quick.shopify.io`

### Local Development

Simply open `index.html` in a browser. Note: BigQuery integration requires Quick Site hosting for OAuth.

## 📈 Dashboard Components

### Charts

1. **📊 Activation Volume by Path** - Stacked bar chart showing monthly activations by path category
2. **📈 BOB Rate by Path** - Line chart showing % in territory at activation time
3. **🌍 Regional Mix** - 100% stacked bar showing AMER/EMEA/APAC distribution
4. **🎯 Plus Leads Self-Serve Rate** - Line chart showing % of new Plus leads routed as 'self-serve' by region

### Stat Cards

- Total Activations
- In BOB (Territory) Count
- Overall BOB Rate %
- Breakdown by Path Category
- Breakdown by Region

### Detailed Table

Full breakdown by Month × Region × Path Category with:
- Total activations
- In BOB count
- BOB rate with visual progress bar

## 🔧 BigQuery Script

The underlying SQL query:

```sql
WITH plus_base AS (
  SELECT
    a.organization_id,
    a.primary_shop_id,
    a.plus_started_on,
    a.plus_trial_started_on,
    a.activation_type,
    a.primary_shop_plus_started_on_gmv_usd_l365d,
    a.primary_shop_country,
    b.contract_type
  FROM `shopify-dw.mart_growth.admin_plus_metrics` a
  LEFT JOIN `shopify-dw.accounts_and_administration.organization_contract_pricing_info_current` b 
    ON a.organization_id = b.organization_id
  WHERE plus_started_on >= '2025-01-01'
    AND plus_started_on < '2026-01-01'
),

skip_trial_shops AS (
  SELECT DISTINCT shop_id
  FROM `shopify-dw.mart_growth.admin_plus_activation_funnel`
),

plus_cohort AS (
  SELECT 
    a.*,
    CASE 
      WHEN a.plus_trial_started_on IS NOT NULL THEN 'Self-serve Trial'
      WHEN b.shop_id IS NOT NULL THEN 'Self-serve Admin'
      ELSE 'Non Self-Serve'
    END AS path_category
  FROM plus_base a
  LEFT JOIN skip_trial_shops b ON a.primary_shop_id = b.shop_id
),

bob_ranked AS (
  SELECT
    pc.organization_id,
    pc.primary_shop_id,
    pc.plus_started_on,
    pc.plus_trial_started_on,
    pc.primary_shop_country,
    pc.activation_type,
    pc.primary_shop_plus_started_on_gmv_usd_l365d,
    pc.contract_type,
    pc.path_category,
    bob.territory_name,
    bob.account_id,
    bob.date,
    ROW_NUMBER() OVER (
      PARTITION BY pc.organization_id
      ORDER BY bob.date DESC
    ) AS rn
  FROM plus_cohort pc
  LEFT JOIN `sdp-stg-commercial.lucasmiller_scratch.account_territory_daily_snapshot` bob
    ON CAST(pc.organization_id AS STRING) = bob.organization_id
    AND bob.date < pc.plus_started_on
),

activation_detail AS (
  SELECT
    br.organization_id,
    br.primary_shop_id,
    br.plus_started_on,
    br.plus_trial_started_on,
    FORMAT_DATE('%Y-%m', br.plus_started_on) AS plus_started_month,
    br.primary_shop_plus_started_on_gmv_usd_l365d,
    br.activation_type,
    br.contract_type,
    br.path_category,
    br.primary_shop_country,
    br.territory_name,
    br.account_id,
    CASE WHEN srr.region IN ('AMER', 'LATAM') OR srr.region IS NULL THEN 'AMER' ELSE srr.region END AS region,
    CASE 
      WHEN br.territory_name IS NULL OR br.territory_name = 'No Territory' THEN 0 
      ELSE 1 
    END AS in_bob
  FROM bob_ranked br
  LEFT JOIN `shopify-dw.accounts_and_administration.business_platform_contracts` bpc
    ON br.organization_id = bpc.organization_id
  LEFT JOIN `shopify-dw.sales.sales_reporting_regions` srr
    ON br.primary_shop_country = srr.country_code
  WHERE br.rn = 1 OR br.rn IS NULL
)

SELECT
  region,
  plus_started_month AS month,
  path_category,
  COUNT(DISTINCT organization_id) AS total_activations,
  COUNT(DISTINCT CASE WHEN in_bob = 1 THEN organization_id END) AS in_bob_count,
  ROUND(100.0 * COUNT(DISTINCT CASE WHEN in_bob = 1 THEN organization_id END) / COUNT(DISTINCT organization_id), 1) AS in_bob_rate_pct
FROM activation_detail
GROUP BY region, plus_started_month, path_category
ORDER BY region, plus_started_month, path_category
```

## 🔐 Permissions Required

- BigQuery read access to the tables listed above
- PII access for `sdp-prd-interim-data-loaders.sensitive_shopify.shops` (if using detailed shop data)

## 📝 Changelog

### 2025-02-04
- Initial release
- Path categorization: Trial, Admin Channel, Sales Assisted
- BOB attribution using territory snapshots
- Regional breakdown: AMER, EMEA, APAC

## 👩‍💻 Author

Tracy Yang (@tracyang0002)

---

Built with 💜 for BOB Attribution Analysis • Powered by Quick + BigQuery

