# Flipkart Complaint Ops Dashboard (Power BI)

**Turning social media complaints into a staffing plan.**

Most social media dashboards answer *"How is our brand perceived?"*
This one answers *"How many people do we need on the refund desk on 6 October?"*

A 6-page Power BI report built on **54,270 Facebook complaint comments** about Flipkart. The comments cover **5 complaint categories** and **7 sale events** in 2025. The report shows that complaints after a sale arrive in **four predictable waves**, and it turns that pattern into a surge roster for each support team.

![Power BI](https://img.shields.io/badge/Power%20BI-F2C811?style=flat&logo=powerbi&logoColor=black)
![DAX](https://img.shields.io/badge/DAX-measures-1F2B8F)
![Power Query](https://img.shields.io/badge/Power%20Query-ETL-217346)
![PBIP](https://img.shields.io/badge/Format-PBIP%20%2B%20TMDL-555)

---

## 📸 Dashboard preview

### 1. Ops Overview
![Ops Overview](1_ops_overview.png)

### 2. Sale Cycle Anatomy
![Sale Cycle Anatomy](2_sale_cycle_anatomy.png)

### 3. The Lag, Proven
![The Lag, Proven](3_the_lag_proven.png)

### 4. Queue Management
![Queue Management](4_queue_management.png)

### 5. Findings & Caveats
![Findings & Caveats](5_findings_and_caveats.png)

📄 Full report as a PDF: [`Flipkart_Complaint_Ops.pdf`](Flipkart_Complaint_Ops.pdf)

---

## 🔑 Key findings

### 1. Complaints arrive in four waves
After a sale opens, each complaint type peaks on a different day, and each type is handled by a different team:

| Wave | Category | Peak day | Avg / day at peak | Team that scales up | Staffing window |
|---|---|---|---|---|---|
| 1 | Pricing | Day +1 | 61 | Comms | Day −1 to +1 |
| 2 | Delivery | Day +4 | 173 | Logistics | Day 0 to +8 |
| 3 | Quality & Fake Product | Day +10/+11 | 51 / 23 | Trust & QC | Day +8 to +11 |
| 4 | Refund | Day +13 | 119 | Refund desk | Day +10 to +15 |

**This is a staffing schedule, not a sentiment report.**

### 2. Refund complaints trail delivery complaints by 8 days
- The median gap between the delivery peak and the refund peak is **8 days** (range 5–10 across all 7 sales).
- Shifting refund complaints back by 8 days gives a correlation of **0.768** with delivery complaints, the highest of any lag tested.
- The shortest gap was **5 days**, during Big Billion Days.

**Action:** the refund desk should staff up about a week after the delivery surge.

### 3. Sales raise volume 4–5×, but not evenly
| Category | Peak vs normal day |
|---|---|
| Pricing | 4.8× |
| Refund | 4.6× |
| Delivery | 4.4× |
| Fake Product | 3.2× |
| Quality | 2.6× |

Pricing complaints spike the hardest but last only about 3 days. They peak on the eve and the opening day of a sale, so **a prepared public statement helps more than extra headcount**.

### 4. 36.4% of complaints get no reply
19,780 comments went unanswered. On a public brand page, this is the first metric to fix, before volume.

### Headline numbers
| Total comments | No-reply rate | Baseline daily | Negative share | Sale days | Delivery + Refund share |
|---|---|---|---|---|---|
| 54,270 | 36.4% | 105 | 75.2% | 44 of 365 | 66.3% |

---

## 🗂️ Report pages

| # | Page | Question it answers |
|---|---|---|
| 1 | **Ops Overview** | How much volume, in which categories, and when? |
| 2 | **Sale Cycle Anatomy** | After a sale opens, when does each complaint type peak? |
| 3 | **The Lag, Proven** | Is the delivery → refund lag real and stable? (what-if shift slider + correlation by lag) |
| 4 | **Queue Management** | When should the desk be staffed? (weekday × hour grid, drill-through to the actual comments) |
| 5 | **Findings & Caveats** | What does the data show, and what can't it show? |
| 6 | **Methodology** | How reliable is the complaint classifier? (confusion matrix) |

---

## 🧱 Data model

```
        Dim_Date (1)
            │  date → date
            ▼
     Fact_Comments (*)

  Dim_SaleCalendar     ← disconnected (lookup only)
  Lag / Shift Days     ← what-if parameter tables
  _Measures            ← all DAX measures, in display folders
```

**Key design choice:** `Dim_SaleCalendar` is a *range* table, where one row covers a 5–9 day sale. Power BI relationships only join on equality, so the sale calendar is **not related** to anything. Instead, calculated columns flatten it into `Dim_Date`:
- `Sale Cycle Name`
- `Days From Sale Start`
- `Days Window`
- `Is Sale Day`

This lets every visual line up on "day 0 = sale opens".

`summary_daily_by_category.csv` is a pre-pivoted copy of the fact table. It is **not loaded**, which avoids having two fact tables at different grains that could disagree. It is kept only to cross-check the measures.

### Tables
| Table | Rows / grain | Purpose |
|---|---|---|
| `Fact_Comments` | 1 row per comment | comment text, category, sentiment, likes, replies, timestamp |
| `Dim_Date` | 1 row per day (2025) | calendar + flattened sale-cycle columns |
| `Dim_SaleCalendar` | 7 sales | start/end dates (disconnected) |
| `Shift Days` | 0–14 | what-if slider for shifting refund comments |
| `Lag` | lag buckets | x-axis for the correlation-by-lag chart |

---

## 🧮 DAX highlights

The model has 100+ measures, organised into display folders (Sale Cycle, Lag, Queue, KPI cards, conditional-format colours).

**Days from sale start** (calculated column that replaces a range relationship):
```dax
Days From Sale Start =
VAR CurrentDate = 'Dim_Date'[date]
VAR PriorSaleStart =
    MAXX (
        FILTER ( 'Dim_SaleCalendar', 'Dim_SaleCalendar'[start_date] <= CurrentDate ),
        'Dim_SaleCalendar'[start_date]
    )
RETURN
    IF ( ISBLANK ( PriorSaleStart ), BLANK (), DATEDIFF ( PriorSaleStart, CurrentDate, DAY ) )
```

**Pearson correlation at a chosen lag** (Delivery today vs Refund *n* days later):
```dax
Correlation at Shift =
VAR lag  = [Shift Value]
VAR maxD = CALCULATE ( MAX ( 'Dim_Date'[date] ), ALL ( 'Dim_Date' ) )
VAR t =
    ADDCOLUMNS (
        FILTER ( VALUES ( 'Dim_Date'[date] ), 'Dim_Date'[date] + lag <= maxD ),
        "@x", [Delivery Comments] + 0,
        "@y",
            VAR d = 'Dim_Date'[date] + lag
            RETURN CALCULATE ( [Refund Comments] + 0, REMOVEFILTERS ( 'Dim_Date' ), 'Dim_Date'[date] = d )
    )
VAR mx  = AVERAGEX ( t, [@x] )
VAR my  = AVERAGEX ( t, [@y] )
VAR cov = SUMX ( t, ( [@x] - mx ) * ( [@y] - my ) )
VAR sx  = SUMX ( t, ( [@x] - mx ) ^ 2 )
VAR sy  = SUMX ( t, ( [@y] - my ) ^ 2 )
RETURN IF ( NOT ISBLANK ( lag ), DIVIDE ( cov, SQRT ( sx * sy ) ) )
```

**Surge multiplier:**
```dax
Surge Multiplier = DIVIDE ( [Peak Value], [Baseline Daily Comments] )
```

---

## 🧪 Methodology: validating the classifier

A keyword-rule classifier was applied to every comment and checked against the supplied labels.

| Metric | Value |
|---|---|
| Comments scored | 54,270 |
| Rule accuracy vs supplied label | **75.5%** |
| Unclassified (no keyword matched) | 20.8% |

- **Rule order:** Fake Product → Refund → Pricing → Delivery → Quality. When a comment matches more than one rule, the rarer and more serious label wins. A counterfeit report sent to the refund desk would lose its trust-and-safety signal.
- **Watch-out:** some Delivery comments get predicted as Refund because they mention both. This is the delivery → refund cascade showing up as classifier noise.

---

## ⚠️ Limitations

- **Sample data:** this is a practice dataset, not Flipkart's internal data. The project shows the method, not real company figures.
- **Sample size:** one year and seven sales is enough to see the pattern but not enough to model it. Treat 8 days as an *observed median*, not a forecast.
- **Sentiment:** only Negative and Neutral values exist. Negative share describes what the corpus contains, not brand health.
- **Products:** all 20 products fall within a 5.8–7.7% fake-product band and have near-identical complaint mixes. No product is riskier than another.
- **Data quality:** no nulls, no duplicate `comment_id`, and `order_id` is unique across all rows.

---

## 📁 Repository structure

```
├── Flipkart_Complaint_Ops/                  # Power BI Project (PBIP), source-control friendly
│   ├── Flipkart_Complaint_Ops.pbip          # ← open this in Power BI Desktop
│   ├── flipkart-ops-theme.json              # custom report theme
│   ├── Flipkart_Complaint_Ops.SemanticModel/  # model as TMDL (tables, measures, relationships)
│   └── Flipkart_Complaint_Ops.Report/         # report pages & visuals as JSON
├── Dataset/
│   ├── fact_comments.csv
│   ├── dim_date.csv
│   ├── dim_sale_calendar.csv
│   ├── summary_daily_by_category.csv        # cross-check only (not loaded)
│   └── Flipkart_Complaint_Taxonomy_Dataset.xlsx
├── BI.pbix                                  # single-file version of the report
├── Flipkart_Complaint_Ops.pdf               # exported report
├── Flipkart_Complaint_Ops_Dashboard_PowerBI_Guide.docx  # step-by-step build guide
└── *.png                                    # page screenshots
```

---

## ▶️ How to open

1. Install the latest **Power BI Desktop** (Windows).
2. Clone the repo:
   ```bash
   git clone https://github.com/<your-username>/<repo-name>.git
   ```
3. Open **`BI.pbix`** for the quick version, or **`Flipkart_Complaint_Ops/Flipkart_Complaint_Ops.pbip`** for the source-controlled project. Using PBIP may require turning on *Power BI Project (.pbip) save option* under **File → Options → Preview features**.
4. If you are asked for data source paths, point them at the `Dataset/` folder (**Transform data → Data source settings → Change source**) and click **Refresh**.

---

## 🛠️ Tools & skills

- **Power BI Desktop:** report design, custom theme, bookmarks/drill-through, what-if parameters
- **Power Query (M):** data loading, type handling, shaping
- **DAX:** calculated columns, time-offset logic, Pearson correlation, surge ratios, dynamic titles, conditional formatting measures
- **Data modelling:** star schema, disconnected range tables, single-direction relationships
- **PBIP / TMDL:** Git-friendly Power BI project format

---

## 👤 Author

**<Your Name>**
[LinkedIn](https://www.linkedin.com/in/<your-profile>) · [GitHub](https://github.com/<your-username>)

If you found this useful, consider giving the repo a ⭐.
