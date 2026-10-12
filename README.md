# Retail Sales Trend Analysis and Future Sales Prediction System

A Django web application that analyses retail sales history (Kaggle **Superstore Sales Dataset**, `train.csv`),
shows trends, seasonality and top performers, and **forecasts future sales with machine learning**.

![sample charts](docs/sample_charts.png)

## Modules

| # | Module | Where | What it does |
|---|--------|-------|--------------|
| 1 | **Sales Data Management** | `/data/` | Import CSV/ZIP (cleaned with Pandas), browse, search, filter, sort, paginate, add / edit / delete records, export to CSV, wipe data |
| 2 | **Sales Trend Analysis** | `/trends/` | Yearly growth, monthly trend, quarterly, seasonality heat-map + seasonal index, weekday pattern, category / sub-category, top & bottom products, regions / states / cities, segments, ship modes, top customers, auto-generated findings |
| 3 | **Future Sales Prediction** | `/predict/` | Compares 4 Scikit-learn models against a baseline, picks the best, forecasts 3-24 months for the whole business or for any category / region / segment, saves the model (`.joblib`) |
| 4 | **Sales Dashboard** | `/` | KPI cards, key charts and insights; filter by year, region, category, segment |

## Technology stack

Frontend HTML/CSS/Bootstrap 5 - Backend Python + Django - Data Pandas, NumPy - Charts Matplotlib, Seaborn -
ML Scikit-learn - Database SQLite (MySQL config included) - IDE VS Code.

## Setup (5 minutes)

```bash
cd retail_sales_project
python -m venv venv
venv\Scripts\activate            # Windows     |  source venv/bin/activate   (macOS / Linux)
pip install -r requirements.txt

python manage.py makemigrations sales
python manage.py migrate
python manage.py import_sales              # loads data/train.csv (9,800 rows)
python manage.py createsuperuser           # optional, for /admin/
python manage.py runserver
```

Open <http://127.0.0.1:8000/>. (You can also import from the browser: **Data Management -> Import CSV**;
a `.zip` containing the CSV works too.)

Useful commands

```bash
python manage.py train_model --horizon 12                # metrics table + saves ml_models/best_model_all_sales.joblib
python manage.py train_model --category Technology
python manage.py test                                    # unit + integration tests
```

## Project structure

```
retail_sales_project/
├── manage.py  requirements.txt  README.md
├── data/train.csv                       # Kaggle dataset
├── config/                              # settings, root urls, wsgi/asgi
├── ml_models/                           # saved .joblib models
└── sales/
    ├── models.py  forms.py  admin.py  views.py  urls.py
    ├── services/
    │   ├── data_loader.py               # read + clean + bulk import        (Module 1)
    │   ├── analysis.py                  # Pandas/NumPy analytics, insights  (Module 2)
    │   ├── charts.py                    # Matplotlib/Seaborn -> base64 PNG  (Modules 2, 3, 4)
    │   └── prediction.py                # feature engineering, ML, forecast (Module 3)
    ├── management/commands/             # import_sales, train_model
    ├── templates/sales/                 # Bootstrap pages
    ├── static/sales/css/style.css
    └── tests/                           # service tests + Django view tests
```

The `services/` package has no Django imports (except the one DB call in `data_loader.import_dataframe`), so the analysis
and ML code can be reused in a Jupyter notebook or another project.

## Dataset notes (checked against the supplied `train.csv`)

* 9,800 order lines, 4,922 orders, 793 customers, 1,861 products, **3 Jan 2015 - 30 Dec 2018**.
* Dates are **`dd/mm/yyyy`** - the importer detects the format automatically (day-first, then month-first, then ISO).
* 11 rows have no Postal Code (stored blank).
* **This file has no `Profit`, `Quantity` or `Discount` column** - only `Sales`. The system therefore analyses *sales*.
  The database, importer and pages already support these columns: if you import a file that contains them (for example the
  original "Sample - Superstore" data), profit, margin and lowest-margin insights appear automatically.

## How the prediction works

1. Transactions are aggregated to a **monthly** series (48 months).
2. Each month is expressed relative to the **trailing 12-month average**, so models can handle the upward trend.
   Features: lag-1/2/3/12, 3-month mean, month sin/cos, and a seasonal index learnt from history.
3. Models: Linear Regression, Ridge, Random Forest, Gradient Boosting - compared with a **rolling-origin back-test**
   (4 folds x 3-month horizon, recursive multi-step, i.e. exactly how the future forecast is produced) against a
   "same month last year" baseline.
4. The lowest-RMSE model is refit on all data and forecasts N months ahead, with an approximate 95% range (+-1.96 x RMSE).

Results on the supplied data (last 12 months as back-test):

| Model | MAE | RMSE | MAPE |
|-------|----:|-----:|-----:|
| **Ridge Regression** (selected) | 12,122 | 14,344 | 21.2% |
| Random Forest | 13,082 | 15,077 | 22.6% |
| Gradient Boosting | 13,596 | 15,634 | 26.1% |
| Linear Regression | 12,542 | 15,807 | 21.3% |
| Seasonal Naive baseline | 15,444 | 18,932 | 24.9% |

Forecast for 2019 ≈ **$824K** (+14% vs 2018's $722K, in line with the 14.6% historical annual growth).

**Honest limitations:** only 48 months of history and very spiky months (e.g. Nov 2018 = 118K vs Jan 2018 = 43K) mean
a ~21% monthly error is realistic. Use forecasts for planning and inventory direction, not exact targets. The page warns you
if a chosen model ever fails to beat the baseline, and refuses scopes with fewer than 30 months of data.

## Key findings in the data

* Sales grew from $480K (2015) to $722K (2018); 2016 dipped 4.3%, then +30.6% and +20.3%.
* Strong seasonality: **Nov, Dec and Sep** are peak months; Q4 contributes ~39% of sales; Jan-Feb are weakest.
* **Technology** is the largest category (36.6%); best sub-category **Phones** ($328K); weakest **Fasteners** ($3K).
* **West** leads regions (31.4%), **South** is smallest (17.2%).
* Top product: *Canon imageCLASS 2200 Advanced Copier* ($61.6K). The top 10 customers give only 6.8% of revenue (low concentration).

## Switching to MySQL

Install `mysqlclient`, create the database, then uncomment the MySQL block in `config/settings.py` (see comments there)
and re-run `migrate` + `import_sales`.

## Troubleshooting

* *Charts blank / Bootstrap unstyled* - the pages load Bootstrap from a CDN, so internet access is needed.
* *"Import failed: Missing required column"* - the CSV must follow the Kaggle Superstore layout (see **Data Management -> Import**).
* *`No module named ...`* - activate the virtual environment and re-run `pip install -r requirements.txt`.
