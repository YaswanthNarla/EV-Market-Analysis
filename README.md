# EV Market Pricing Analysis

A data pipeline that scrapes, cleans, and analyses electric vehicle listings to surface what actually drives EV prices — and where the market gaps are.

---

## Why This Exists

EV pricing makes no intuitive sense on the surface.

Two vehicles, similar size, similar year. One costs ₹30 lakhs. The other costs ₹80 lakhs. A buyer browsing listings can't immediately explain the gap. A business entering the EV space can't afford not to.

The pricing signals are buried in the data — battery capacity, range, brand positioning, segment clustering. This project pulls them out. It started as a question: *what actually determines what an EV costs?* It ended as a working answer backed by 10,000+ scraped records.

---

## What This Project Does

Scrapes EV listing data from the web, cleans the raw mess that web scraping always produces, and runs correlation and multivariate analysis to identify the real pricing drivers.

The output isn't a dashboard. It's a set of findings that a product team, pricing strategist, or market analyst can act on directly.

---

## Dataset

| Property | Detail |
|---|---|
| Source | Web-scraped EV listings |
| Records | 10,000+ |
| Collection Method | Python (Requests + BeautifulSoup) |
| Core Features | Brand, Battery Capacity (kWh), Driving Range (km), Price |

Raw data came in the way scraped data always does — inconsistent formatting, mixed units, missing values, brand names with typos, prices in different currencies or with currency symbols baked in. The cleaning pipeline handled all of it before a single analysis ran.

---

## Project Structure

```
ev_market_analysis/
│
├── data/
│   ├── raw/
│   │   └── ev_listings_raw.csv          # Scraped output, untouched
│   └── processed/
│       └── ev_listings_clean.csv        # After cleaning pipeline
│
├── notebooks/
│   ├── 01_scraping.ipynb                # Data collection pipeline
│   ├── 02_cleaning.ipynb                # Cleaning & structuring
│   └── 03_eda.ipynb                     # EDA, correlation, insights
│
├── src/
│   ├── scraper.py                       # Requests + BeautifulSoup pipeline
│   └── clean.py                         # Data cleaning functions
│
├── visuals/
│   └── *.png                            # Saved plots from EDA
│
├── requirements.txt
└── README.md
```

---

## How I Built It

### Stage 1 — Scraping

Built a data pipeline using Python's `Requests` library for HTTP calls and `BeautifulSoup` for HTML parsing.

The scraper iterated through paginated listing pages, extracted the relevant fields per vehicle — brand, battery capacity, range, and price — and wrote raw output to CSV after each batch. Pagination handled, delays between requests to avoid rate-limiting, exception handling for pages that returned incomplete records.

Web scraping is rarely clean on the first pass. The raw file reflected that — mixed formats, inconsistent field names, some records with partial data, others with HTML artifacts still attached to string fields.

```python
import requests
from bs4 import BeautifulSoup
import pandas as pd
import time

def scrape_ev_listings(base_url, pages):
    records = []
    for page in range(1, pages + 1):
        response = requests.get(f"{base_url}?page={page}", timeout=10)
        soup = BeautifulSoup(response.content, "html.parser")
        listings = soup.find_all("div", class_="ev-listing-card")
        for listing in listings:
            records.append({
                "brand"  : listing.find("span", class_="brand").text.strip(),
                "battery": listing.find("span", class_="battery").text.strip(),
                "range"  : listing.find("span", class_="range").text.strip(),
                "price"  : listing.find("span", class_="price").text.strip(),
            })
        time.sleep(1.2)
    return pd.DataFrame(records)
```

### Stage 2 — Cleaning

Raw scraped data doesn't go into analysis. It goes into a cleaning pipeline first.

What that meant here: stripping currency symbols and commas from price fields and converting to float, extracting numeric values from strings like `"75 kWh"` and `"420 km"`, standardising brand names (`"Tesla"`, `"TESLA"`, `"tesla "` all became `"Tesla"`), dropping records missing more than one core field, and flagging statistical outliers in price for review before deciding whether to keep or cap them.

After cleaning: structured, typed, consistent. Every column had the data type it was supposed to have, and the brand field had exactly one spelling per brand.

```python
import pandas as pd
import re

def clean_price(val):
    val = re.sub(r'[₹$,\s]', '', str(val))
    return float(val) if val else None

def extract_numeric(val):
    match = re.search(r'[\d.]+', str(val))
    return float(match.group()) if match else None

df['price']   = df['price'].apply(clean_price)
df['battery'] = df['battery'].apply(extract_numeric)
df['range']   = df['range'].apply(extract_numeric)
df['brand']   = df['brand'].str.strip().str.title()
df.dropna(subset=['price', 'battery', 'range', 'brand'], inplace=True)
```

### Stage 3 — EDA & Analysis

With clean data, the analysis ran across three layers.

**Univariate:** Distribution of price, battery capacity, and range. Price was right-skewed — most listings clustered in the mid-range with a long tail of premium vehicles. Battery capacity showed a bimodal distribution: a cluster around 40–60 kWh (mass-market EVs) and another around 80–100 kWh (premium segment).

**Bivariate:** Price against battery capacity, price against range, and price broken out by brand. The correlation between battery and price was strong. Range and price moved together, but with more noise — brand positioning diluted a purely technical relationship.

**Multivariate:** Pairplot across all numerical features coloured by brand tier (premium vs mass-market). Heatmap of correlation coefficients. Scatter overlays showing how brand identity shifts an EV's price even when battery and range are held roughly constant — a premium badge adds a measurable premium regardless of specs.

---

## Key Findings

**Battery capacity is the strongest pricing signal.**
Across the full dataset, kWh capacity correlated more tightly with final price than any other single feature. Every 10 kWh increase in battery size corresponded to a consistent step up in price within each brand segment.

**Range matters, but brand mediates it.**
Driving range and price correlate strongly in aggregate. But within a brand tier, a mass-market EV with 400km range prices significantly below a premium EV with the same range. The technical spec isn't enough — market positioning shapes what a buyer expects to pay before they ever look at the numbers.

**Premium brands own the high end, but not the volume.**
The top price segment — roughly the top 15% of listings by price — is dominated by three to four brand names. Those brands represent a small fraction of total listings. The volume sits in the middle.

**Mid-range EVs show the highest demand concentration.**
The 300–450km range bracket at 50–70 kWh battery capacity had the highest density of listings and, by implication, the highest market activity. This is where competition is most intense and where price differentiation is hardest to sustain.

**The competitive gap lives at the high-range, mid-price intersection.**
There are relatively few EVs offering 450km+ range at a non-premium price point. That's either a technical constraint (large batteries cost more) or a market opportunity, depending on who's reading this.

---

## Business Impact

**For pricing strategy:** Battery capacity and range should anchor any pricing model for a new EV entrant. Pricing purely on brand positioning without matching the spec expectations of the target segment will compress margins or kill demand.

**For market positioning:** The mid-range segment is crowded. A new entrant competing on specs alone in that space faces significant differentiation pressure. The cleaner opportunity is the 400–500km range at accessible price points — underserved relative to buyer interest.

**For competitive analysis:** Premium brands command a price premium that isn't fully explained by battery or range. That gap is brand equity. Quantifying it (which this analysis does implicitly) tells a new brand what it needs to spend on positioning versus product to compete in each segment.

---

## Tools Used

| Tool | Purpose |
|---|---|
| `Requests` | HTTP calls to listing pages |
| `BeautifulSoup` | HTML parsing |
| `Pandas` | Data structuring and cleaning |
| `Matplotlib` | Distribution and scatter plots |
| `Seaborn` | Correlation heatmap, pairplots |
| `NumPy` | Numerical operations |

---

## Setup

```bash
git clone https://github.com/<your-username>/ev-market-analysis.git
cd ev-market-analysis
pip install -r requirements.txt
```

**Run the scraper:**
```bash
python src/scraper.py
```

**Run the cleaning pipeline:**
```bash
python src/clean.py
```

**Open the analysis notebooks:**
```bash
jupyter notebook notebooks/03_eda.ipynb
```

**`requirements.txt`**
```
requests
beautifulsoup4
pandas
numpy
matplotlib
seaborn
jupyter
```

---

## What I'd Build Next

A price estimator — input brand, battery, range, get a predicted market price with a confidence interval. The correlation structure in this data is strong enough to support a simple regression model that would be genuinely useful for anyone trying to benchmark a listing.

Automated re-scraping on a schedule so the dataset stays current. EV pricing moves fast right now. A static dataset ages faster than most.

Brand tier classification to formalise what the EDA showed intuitively — premium, mass-market, and emerging — so comparisons can be made systematically rather than by eye.

---

## Author

Yaswanth — Data Analyst & ML Practitioner, Hyderabad.
Innomatics Research Labs · AWS Solutions Architect Associate

---

*Web scraping is just the start. The value is in knowing what to look for once the data is clean.*
