# E-Commerce Returns & RTO Cost Leakage Analytics

An enterprise-grade, decision-oriented analytics case study investigating reverse logistics cost leakage in Indian e-commerce marketplace operations (modeled at Flipkart scale). This project moves beyond generic reporting metrics to isolate specific operational failure points, evaluate trade-offs between **prevention** and **cure** strategies, and provide dynamic financial what-if scenario modeling.

---

## Executive Summary & Business Problem

In multi-category e-commerce marketplaces, reverse logistics operates as an uncontrolled margin drain. Most operational reporting lumps all returns together into a single "return rate" KPI. This project decouples reverse logistics into two distinct operational failure modes across **12,000+ orders** and **2,969 return events** representing **₹15.36L** in total annual leakage:

```
                            FORWARD FULFILLMENT FUNNEL
                                         │
                          ┌──────────────┴──────────────┐
                          ▼                             ▼
                  [CUSTOMER RTO]               [CUSTOMER RETURN]
                 (Undelivered)                    (Delivered)
                          │                             │
            • Refused at door / bad address  • Accepted, then shipped back
            • Zero refund issued             • Full refund issued
            • Forward + Reverse freight      • Reverse freight + QC + Restock
            • Pure operational loss          • Product devaluation / Write-off

```

### Core Strategic Findings

1. **The Frequency vs. Severity Paradox:** **Fashion** exhibits the highest return rate (**29.54%**), but its average loss per event is low (**₹289.25**). Conversely, **Furniture** drives **48.94% of total dollar leakage (₹7.52L)** despite a lower return rate (**19.53%**), because heavy weight classes trigger **₹1,202.61** in average loss per event.
* **Action:** Fashion requires **Prevention-Side Interventions** (sizing guides, listing accuracy), whereas Furniture requires **Cure-Side Interventions** (regional fulfillment hubs to reduce reverse shipping distances).


2. **Debunking the "COD Myth":** Market intuition assumes Cash-on-Delivery (COD) customers return goods more frequently. Disaggregating by return type reveals that for items actually delivered and accepted, **Prepaid customers return products at a higher rate than COD customers (15.52% vs. 13.13%)**. COD’s excess failure is **100% concentrated in forward RTOs (16.94% vs. Prepaid's 3.43%)**—making it a forward-delivery execution issue (address verification, customer reachability) rather than a product satisfaction problem.

---

## Repository Structure

```
├── data/
│   ├── raw/                       # Original transactional extracts
│   │   ├── customers.xlsx          # Customer CRM profiles & city tiers
│   │   ├── products.xlsx           # Product catalog, MRP, weight classes
│   │   ├── orders.xlsx             # Order headers & payment methods
│   │   ├── order_items.xlsx        # Line-item details & unit prices
│   │   ├── returns.xlsx            # Event log (RTO vs. Customer Return)
│   │   └── cost_reference.xlsx     # Operational unit costs by category
│   └── processed/
│       └── cleaned_database.db     # SQLite/PostgreSQL cleaned schema export
├── sql/
│   ├── 01_data_cleaning.sql        # String standardization & deduplication
│   ├── 02_loss_attribution.sql     # Master SQL economic loss calculation
│   └── 03_exploratory_cuts.sql     # Category, Payment, & Temporal aggregations
├── excel/
│   └── reverse_logistics_cost_model.xlsx  # Live formula-driven financial model
├── powerbi/
│   └── reverse_logistics_dashboard.pbix   # Interactive Power BI report
├── docs/
│   ├── executive_summary.pdf       # 2-page C-suite business memo
│   └── technical_specifications.md  # Detailed methodology & formulas
└── README.md                       # Project documentation

```

---

## Data Schema & Relational Model

The underlying dataset mirrors fragmented enterprise database systems (OMS, CRM, WMS, and Finance):

```
[orders] ─── (order_id) ───> [order_items] <─── (product_id) ─── [products]
   │                               │                                   │
(customer_id)                 (order_item_id)                      (category)
   │                               │                                   │
   ▼                               ▼                                   ▼
[customers]                    [returns] ──────────────────────> [cost_reference]

```

| Table Name | Entity Granularity | Key Primary/Foreign Keys | Critical Analytics Fields |
| --- | --- | --- | --- |
| `customers` | 1 row / Customer | `customer_id` | `city_name`, `customer_state`, `city_tier` |
| `products` | 1 row / SKU | `product_id` | `category`, `sub_category`, `mrp`, `weight_class` |
| `orders` | 1 row / Order | `order_id`, `customer_id` | `order_date`, `payment_method`, `order_status` |
| `order_items` | 1 row / Line Item | `order_item_id`, `order_id`, `product_id` | `quantity`, `unit_price` |
| `returns` | 1 row / Return Event | `return_id`, `order_item_id` | `return_type`, `return_date`, `refund_amount`, `reason` |
| `cost_reference` | 1 row / Category | `category` | `reverse_shipping_cost_inr`, `qc_inspection_cost_inr`, `restocking_cost_inr`, `avg_markdown_pct_of_price`, `avg_writeoff_pct_of_returns` |

---

## Key Insights & Analytical Proof

### 1. Category Dynamics: High Frequency vs. High Severity

```
                                  CATEGORY TRADE-OFF MATRIX
  
    High Avg Loss (₹)
         │
  ₹1,400 ┼                                                  [FURNITURE]
         │                                               (₹1,202.61 / 19.53%)
  ₹1,200 ┼
         │
  ₹1,000 ┼
         │
    ₹800 ┼                                  [ELECTRONICS]
         │                               (₹623.42 / 11.56%)
    ₹600 ┼
         │                                  [HOME & KITCHEN]
    ₹400 ┼                                 (₹336.63 / 17.48%)
         │                                                    [FASHION]
    ₹200 ┼                                              (₹289.25 / 29.54%)
         │                                  [BEAUTY]
      ₹0 ┴─────────────────────────────(₹180.62 / 15.02%)────────────────────────────
         0%            5%           10%           15%           20%           25%           30%
                                                                                Return Rate (%)

```

| Category | Total Loss (₹) | % of Total Loss | Avg Loss / Event (₹) | Total Returns | Total Sold | Return Rate (%) | Primary Lever |
| --- | --- | --- | --- | --- | --- | --- | --- |
| **Furniture** | **₹7,51,629.92** | **48.94%** | **₹1,202.61** | 625 | 3,201 | 19.53% | **Cure** (Logistics) |
| **Fashion** | ₹2,60,035.25 | 16.93% | ₹289.25 | **899** | 3,043 | **29.54%** | **Prevention** (Listing/Fit) |
| **Electronics** | ₹2,59,343.09 | 16.89% | ₹623.42 | 416 | 3,600 | 11.56% | Balanced |
| **Home & Kitchen** | ₹1,70,333.14 | 11.09% | ₹336.63 | 506 | 2,895 | 17.48% | Balanced |
| **Beauty & Personal Care** | ₹94,462.20 | 6.15% | ₹180.62 | 523 | 3,482 | 15.02% | Prevention |
| **Total / Overall** | **₹15,35,803.60** | **100.00%** | **₹517.28** | **2,969** | **16,221** | **18.30%** | — |

* **Furniture Analysis:** Furniture AOV (₹7,952) was hypothesis-tested against Electronics AOV (₹8,075). Despite similar ticket prices, Furniture’s loss per return event is **1.93x higher** due to fixed freight costs (`Heavy` class: ₹350 reverse shipping, ₹120 QC, ₹80 restocking).

---

### 2. Payment Method Breakdown: COD vs. Prepaid

```
                  PAYMENT METHOD FAILURE DECOMPOSITION
  
  COD Orders (6,252 baseline)
  ├── RTO Failure Rate: 16.94% (1,059 events) ──────────────► [FORWARD DELIVERY ISSUE]
  └── Customer Return Rate: 13.13% (821 events) ───────────► [LOWER THAN PREPAID]
  
  Prepaid Orders (5,748 baseline)
  ├── RTO Failure Rate: 3.43% (197 events) ───────────────► [NEGLIGIBLE]
  └── Customer Return Rate: 15.52% (892 events) ───────────► [HIGHER THAN COD]

```

| Payment Method | Return Type | Total Rupee Loss (₹) | Avg Loss / Event (₹) | Event Count | Order Baseline | Event Rate (%) |
| --- | --- | --- | --- | --- | --- | --- |
| **COD** | **RTO** | **₹2,36,555.00** | **₹223.38** | **1,059** | 6,252 | **16.94%** |
| **COD** | Customer Return | ₹5,95,818.56 | ₹725.72 | 821 | 6,252 | 13.13% |
| **PREPAID** | **RTO** | ₹45,790.00 | ₹232.44 | 197 | 5,748 | **3.43%** |
| **PREPAID** | Customer Return | ₹6,57,640.04 | ₹737.26 | 892 | 5,748 | **15.52%** |

---

## Methodology & Pipeline Mechanics

### Financial Loss Formula

Net economic loss per return event is evaluated as:

$$\text{Loss}_{\text{event}} = \text{COALESCE}(\text{Refund}, 0) + \text{Cost}_{\text{rev\_ship}} + \text{Cost}_{\text{QC}} + \text{Cost}_{\text{restock}} - \text{Recovery}_{\text{est}}$$

Where estimated product value recovery applies **exclusively to Customer Returns**:

$$\text{Recovery}_{\text{est}} = \begin{cases} \text{Refund} \times \left(1 - \text{Markdown\%} - \text{Writeoff\%}\right), & \text{if Return\_Type} = \text{'Customer Return'} \\ 0, & \text{if Return\_Type} = \text{'RTO'} \end{cases}$$

---

### Core Pipeline Snippets

#### 1. SQL Net Loss Attribution Query

```sql
WITH base_returns_calculated AS (
    SELECT 
        rf.return_id,
        rf.order_item_id,
        rf.return_type,
        -- Handle NULL refunds for RTO events to prevent arithmetic failure
        COALESCE(rf.refund_amount, 0) AS clean_refund_amount,
        COALESCE(rf.reason, 'Reason Not Captured') AS clean_reason,
        oc.payment_method,
        p.category,
        cr.reverse_shipping_cost_inr,
        cr.qc_inspection_cost_inr,
        cr.restocking_cost_inr,
        -- Recovery applies only to Customer Returns
        CASE 
            WHEN rf.return_type = 'Customer Return' 
            THEN COALESCE(rf.refund_amount, 0) * (1 - cr.avg_markdown_pct_of_price - cr.avg_writeoff_pct_of_returns)
            ELSE 0 
        END AS estimated_recovery
    FROM returns rf
    INNER JOIN order_items oi ON oi.order_item_id = rf.order_item_id
    INNER JOIN orders_cleaned oc ON oc.order_id = oi.order_id
    INNER JOIN products p ON p.product_id = oi.product_id
    INNER JOIN cost_reference cr ON cr.category = p.category
)
SELECT 
    category,
    payment_method,
    return_type,
    COUNT(DISTINCT return_id) AS total_events,
    ROUND(SUM(clean_refund_amount + reverse_shipping_cost_inr + qc_inspection_cost_inr + restocking_cost_inr - estimated_recovery), 2) AS total_net_loss_inr
FROM base_returns_calculated
GROUP BY category, payment_method, return_type;

```

#### 2. Excel Formulas (`reverse_logistics_cost_model.xlsx`)

* **Dynamic Cost Lookup:** Uses `INDEX/MATCH` to avoid `VLOOKUP` column-shift fragility:
```excel
=INDEX(Assumptions!$C$2:$C$6, MATCH(D2, Assumptions!$A$2:$A$6, 0))

```


* **Text-to-Date Serial Conversion:** Resolves Excel PivotTable date-grouping failures caused by text strings (`YYYY-MM-DD`):
```excel
=DATEVALUE(H2)

```



#### 3. Power BI DAX Measures

```dax
// Total Net Loss Measure
Total_Net_Loss = 
SUMX(
    returns,
    VAR Refund = COALESCE(returns[refund_amount], 0)
    VAR RevShip = RELATED(cost_reference[reverse_shipping_cost_inr])
    VAR QC = RELATED(cost_reference[qc_inspection_cost_inr])
    VAR Restock = RELATED(cost_reference[restocking_cost_inr])
    VAR MarkdownPct = RELATED(cost_reference[avg_markdown_pct_of_price])
    VAR WriteoffPct = RELATED(cost_reference[avg_writeoff_pct_of_returns])
    
    VAR Recovery = 
        IF(
            returns[return_type] = "Customer Return",
            Refund * (1 - MarkdownPct - WriteoffPct),
            0
        )
    RETURN
        Refund + RevShip + QC + Restock - Recovery
)

// Dynamic What-If Scenario Savings Projection
Projected_Annual_Savings = 
VAR Selected_CODRTO_Reduction = SELECTEDVALUE('COD_RTO_Slider'[COD_RTO_Slider Value], 0.20)
VAR Selected_Fashion_Reduction = SELECTEDVALUE('Fashion_Slider'[Fashion_Slider Value], 0.15)

VAR Baseline_COD_RTO_Loss = 
    CALCULATE([Total_Net_Loss], orders[payment_method] = "COD", returns[return_type] = "RTO")

VAR Baseline_Fashion_Return_Loss = 
    CALCULATE([Total_Net_Loss], products[category] = "Fashion", returns[return_type] = "Customer Return")

RETURN
    (Baseline_COD_RTO_Loss * Selected_CODRTO_Reduction) + 
    (Baseline_Fashion_Return_Loss * Selected_Fashion_Reduction)

```

---

## Business Impact & Scenario Analysis

The workbook houses an interactive **What-If Financial Sensitivity Engine** (`reverse_logistics_cost_model.xlsx`) that projects annualized net recovery based on operational levers:

```
                  SCENARIO ANALYSIS CALCULATION MECHANICS

  Baseline COD RTO Loss (₹2,36,555.00)  ×  Lever A Input (20.0%)  =  ₹47,311.00 Savings
                                                                          │
  Baseline Fashion Loss (₹2,25,595.00)  ×  Lever B Input (15.0%)  =  ₹33,839.25 Savings
                                                                          │
                                                                          ▼
                                                                ₹81,150.25 Total Savings

```

| Operational Intervention Lever | Baseline Loss (₹) | Target Efficiency | Projected Annual Recovery (₹) |
| --- | --- | --- | --- |
| **Lever A:** COD Address Verification & NDR Confirmation | ₹2,36,555.00 | 20% RTO Reduction | **₹47,311.00** |
| **Lever B:** Fashion Sizing & Listing Specification Fixes | ₹2,25,595.00 | 15% Return Reduction | **₹33,839.25** |
| **Combined Portfolio Recovery** | **₹15,35,803.60** | **Multi-Lever** | **₹81,150.25 / year** |

---

## Getting Started

### Prerequisites

* **SQL:** PostgreSQL 13+ or SQLite 3.30+
* **Excel:** Microsoft Excel 2019 / Office 365 (supports Data Tables & Dynamic Arrays)
* **Power BI:** Power BI Desktop (May 2023 release or newer)

### Quick Replication Steps

1. **Clone the Repository:**
```bash
git clone https://github.com/your-username/reverse-logistics-analytics.git
cd reverse-logistics-analytics

```


2. **Execute SQL Attribution Model:**
* Import transactional CSVs into your target database.
* Run `sql/01_data_cleaning.sql` followed by `sql/02_loss_attribution.sql`.


3. **Run the Excel Scenario Model:**
* Open `excel/reverse_logistics_cost_model.xlsx`.
* Modify parameters on the `Scenario_Inputs` tab to observe dynamic recalculations across the workbook.


4. **Explore Power BI Dashboard:**
* Open `powerbi/reverse_logistics_dashboard.pbix` in Power BI Desktop.
* Adjust what-if parameter sliders to inspect real-time DAX measure updates.



---

## License & Contact

This project is licensed under the MIT License - see the `LICENSE` file for details.

* **Author:** Dev Bansal
* **Role:** Business Analyst / Supply Chain Analytics
* **Links:** [LinkedIn](https://www.google.com/search?q=https://linkedin.com/in/) | [GitHub](https://github.com/) | [Portfolio](https://tinyurl.com/)
