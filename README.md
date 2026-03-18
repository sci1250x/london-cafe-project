# London Cafe Mapper

A Streamlit app that maps every cafe chain across London, traces each brand to its parent company, and enriches the data with live stock prices, market cap figures, and ownership structures — all from free or near-free data sources.

---

## What It Does

You load a Google Places API key (or use demo data), and the app:

1. Fetches every cafe in London via a spatial grid search
2. Identifies which chain each cafe belongs to using fuzzy brand matching
3. Looks up each parent company on Wikipedia to find subsidiaries and private valuations
4. Pulls live stock data for listed companies via Yahoo Finance
5. Renders an interactive ownership tree and filterable data table

---

## How It Works — Pipeline Walkthrough

The app is split into two files: **`london_cafe_pipeline.py`** handles all data logic, and **`app.py`** is a pure UI layer that calls into it.

### Stage 1 — Spatial Grid + Google Places API

London is divided into a grid of overlapping circles (~1.8 km radius, ~33% overlap) covering a 28 km span from the centre. For each grid point the app sends a `nearbysearch` request to the Google Places API with `type=cafe`.

```
London (28km span)
└── ~280 grid points
    └── up to 3 pages × 20 results per point
        └── deduplicated by place_id → ~5,000–8,000 unique cafes
```

Results are cached to `data/places_cache.json` after the first run. Every subsequent run is instant and free — the API is only ever called once.

**Cost:** ~$6.60 one-off (fits comfortably within Google's $200/month free tier).

---

### Stage 2 — Smart Brand Normalisation (free, local)

Raw place names from Google Maps are messy: `"Starbucks - Canary Wharf"`, `"Pret | Victoria"`, `"Costa Coffee (Liverpool Street)"`. The normaliser:

1. **Strips location suffixes** via regex — anything after `-`, `|`, `/`, `(`, `at`, `in`, `near`
2. **Exact prefix match** against the curated `CHAIN_KEYWORDS` list
3. **Fuzzy match** using [RapidFuzz](https://github.com/maxbachmann/RapidFuzz) `token_sort_ratio` — handles word-order variation (`"Nero Caffè"` → `"Caffè Nero"`)
4. **Partial ratio fallback** on the original name in case stripping removed too much

A `BRAND_CANONICAL` dictionary then collapses variants to their official name (`"Pret"` → `"Pret A Manger"`, `"Watch House"` → `"WatchHouse"`, etc.).

If no match is found the cafe is treated as an independent — it appears in the table but not the ownership tree.

---

### Stage 3 — Parent Company Lookup

Every recognised brand maps to a `(parent_company, ticker)` pair in `BRAND_TO_PARENT`. For example:

| Brand | Parent | Ticker |
|---|---|---|
| Costa Coffee | The Coca-Cola Company | KO |
| Pret A Manger | JAB Holding Company | — |
| Greggs | Greggs plc | GRG.L |
| WatchHouse | WatchHouse Coffee Ltd | — |
| Leon | Leon Restaurants Ltd | — |

Chains owned by the same parent (e.g. Pret and EAT both under JAB Holding) automatically group together in the ownership tree.

---

### Stage 4 — Wikipedia Single Pass (free)

For every unique parent company, the app makes **one batch HTTP request** to the Wikipedia API that fetches both:

- **Subsidiaries** — parsed from the `subsidiaries` infobox field
- **Private valuation estimate** — parsed from `revenue`, `assets`, `equity` or `valuation` infobox fields (used for private companies where yfinance has no data)

This is designed to be a single call per `WIKI_BATCH` (10) companies, so ~15 parent companies = **2 total HTTP requests**. Results are cached to `data/wiki_cache.json`.

---

### Stage 5 — Live Stock Data via yfinance (free)

For every parent with a valid ticker, the app calls [yfinance](https://github.com/ranaroussi/yfinance) to fetch:

- Current price (converted to USD via live FX rates)
- Market cap
- P/E ratio
- Sector and industry
- 52-week high/low

Private companies get their valuation estimate from Wikipedia instead.

---

### Stage 6 — Entity Type Classification (free, local)

Every brand and Wikipedia subsidiary is classified into a category using keyword matching against its name — no extra API calls required:

| Category | Examples |
|---|---|
| Cafe / Coffee | Starbucks, Teavana, Nespresso, espresso-named brands |
| Bakery | Gail's, Paul Bakery, Patisserie Valerie |
| Sandwich / Deli | Pret A Manger, EAT, Upper Crust |
| Fast Food | McDonald's, Burger King, Popeyes |
| Restaurant | Leon, Wagamama |
| Juice / Health | Crussh, Benugo Juice |
| Retail / Other | Holdings, media, entertainment entities |
| Food & Beverage | Fallback for anything unmatched |

The `entity_type` column appears in the data table. In the visual map, subsidiaries classified as **Cafe / Coffee** receive the same bold blue underline as actual London cafe locations — all other subsidiaries get a hue-coloured underline based on their parent's industry.

---

### Stage 7 — Final DataFrame

Everything is joined into a single enriched DataFrame — one row per cafe location — with columns for brand, entity type, parent, ticker, price, market cap, rating, review count, and location count. This is the source of truth for both the table and the visual mapping.

---

## Visual Mapping

The ownership tree is rendered using [Markmap](https://markmap.js.org/) (a D3-based mind-map library) embedded in a Streamlit HTML component.

- The root is `# London Cafes`
- Listed parents show as `## Starbucks (SBUX $234.56)` — ticker and live price
- Private parents show as `## Caffè Nero (PRIVATE)`
- Brands that ARE their parent (e.g. Greggs/Greggs plc) collapse to a single node — no redundant parent edge
- Real ownership edges (e.g. `McCafé → McDonald's Corporation`, `Costa Coffee → Coca-Cola`) show the parent as the `##` node with brands as `###` children
- Ticker/price text is rendered at 70% size and 52% opacity so it doesn't compete with the brand name
- **Bold blue underline** = actual London cafe locations (≥50% of Google Places records tagged `cafe`) OR subsidiaries classified as Cafe / Coffee
- **Coloured underline** = non-cafe subsidiaries — colour determined by parent industry (see legend)
- **No underline** = parent company nodes
- Industry legend is shown at the right-middle of the map

Pan with click-drag, zoom with scroll or the +/− buttons, and click any node to expand/collapse.

---

## Knowledge Base

The app recognises **50+ London cafe chains** including:

**International chains:** Starbucks, Costa Coffee, Caffè Nero, Pret A Manger, Greggs, McCafé, Tim Hortons, Nespresso, Blue Bottle, Upper Crust, Le Pain Quotidien, Lavazza, Paul Bakery, Itsu

**London/UK chains:** Gail's Bakery, Leon, Black Sheep Coffee, Benugo, AMT Coffee, Blank Street Coffee, Grind, Caravan, WatchHouse, Daisy Green, Notes Coffee, Workshop Coffee, Allpress Espresso, Ozone Coffee, Ole & Steen, Joe & the Juice, Federation Coffee, Redemption Roasters, Origin Coffee, Milk Beach, Vagabond, Crussh, Harris + Hoole, Patisserie Valerie, Boston Tea Party

New chains can be added by appending entries to `CHAIN_KEYWORDS`, `BRAND_TO_PARENT`, and (if needed) `BRAND_CANONICAL` in `london_cafe_pipeline.py`.

---

## Installation & Running

### Prerequisites

- Python 3.9+
- A Google Places API key (optional — demo data works without one)

### Setup

```bash
git clone https://github.com/sci1250x/london-cafe-project.git
cd london-cafe-project
pip install -r requirements.txt
streamlit run app.py
```

### Getting a Google Places API key

1. Go to [console.cloud.google.com](https://console.cloud.google.com)
2. Create a project → enable the **Places API**
3. Create an API key under **Credentials**
4. Google gives $200/month free — the full London fetch costs ~$6.60

### First run

- Paste your API key and click **Load Data** — this fetches and caches all London cafes (~2–5 min)
- Or click **Load Demo Data** to explore instantly with 19 sample records
- Enrichment (Wikipedia + yfinance) runs automatically after loading and takes ~30–60 seconds
- All data is cached locally — subsequent runs are instant

---

## Project Structure

```
london-cafe-project/
├── app.py                    # Streamlit UI — no data logic here
├── london_cafe_pipeline.py   # All pipeline stages: fetch, normalise, enrich
├── requirements.txt
└── data/
    ├── places_cache.json     # Cached Google Places results (created on first run)
    └── wiki_cache.json       # Cached Wikipedia results (created on first run)
```

---

## Tech Stack

| Tool | Purpose |
|---|---|
| [Streamlit](https://streamlit.io) | Web app framework |
| [Google Places API](https://developers.google.com/maps/documentation/places) | Cafe location data |
| [RapidFuzz](https://github.com/maxbachmann/RapidFuzz) | Fuzzy brand name matching |
| [Wikipedia API](https://en.wikipedia.org/w/api.php) | Subsidiaries + private valuations |
| [yfinance](https://github.com/ranaroussi/yfinance) | Live stock prices and market caps |
| [Markmap](https://markmap.js.org/) | Interactive ownership tree visualisation |
| [D3.js](https://d3js.org/) | Pan/zoom behaviour on the map |
| pandas | Data wrangling throughout the pipeline |

---

## Caching Strategy

The app is designed to be cheap to run and fast after the first load:

| Cache | File | Cost | Reset |
|---|---|---|---|
| Google Places | `data/places_cache.json` | ~$6.60 once | Clear Cache button |
| Wikipedia | `data/wiki_cache.json` | Free | Delete the file |
| yfinance | In-memory per session | Free | App restart |

Stock prices refresh every session (they're not persisted) so financials are always live.
