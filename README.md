# London Cafe Mapper

A Streamlit app that maps every cafe chain across London, traces each brand to its parent company, and enriches the data with live stock prices and market cap figures.

## Features
- Full London coverage via Google Places API (cached after first run — ~$6.60 one-off cost)
- Smart brand normalisation — "Pret - Paddington" resolves to "Pret A Manger", etc.
- Ownership tree: each brand traced to its listed or private parent group
- Live financials via yfinance (price, market cap, P/E) for listed parents
- Private company valuation estimates from Wikipedia and Crunchbase
- Interactive markmap visualisation with pan, zoom, and expand/collapse
- Filter by brand, parent company, and listing type (listed vs private)
- CSV export

## Installation
1. Clone the repo
2. Install dependencies: `pip install -r requirements.txt`
3. Run: `streamlit run app.py`

## Usage
- Paste a Google Places API key to fetch live London cafe data (~$6.60, cached forever after)
- Or click **Load Demo Data** to explore without an API key
- Get a free API key at [console.cloud.google.com](https://console.cloud.google.com) ($200/month free tier)

## Data Sources
- [Google Places API](https://developers.google.com/maps/documentation/places/web-service) — cafe locations
- [Wikipedia](https://en.wikipedia.org) — subsidiaries and private valuation estimates
- [yfinance](https://github.com/ranaroussi/yfinance) — live stock data for listed parents
- [Crunchbase](https://www.crunchbase.com) — private company funding data (free tier, no key needed)
