<p align="center"><a href="https://apify.com/DevilScrapes/mercadolibre-scraper"><img src="assets/banner.svg" alt="Mercado Libre Search Results Scraper — Python example" width="100%"></a></p>

# Mercado Libre Scraper in Python: Mexico search results as a pandas-ready table

<p align="center">
<a href="https://apify.com/DevilScrapes/mercadolibre-scraper"><img alt="Run on Apify" src="https://img.shields.io/badge/Run%20on-Apify-DC2626?logo=apify&logoColor=white"></a>
<img alt="Python" src="https://img.shields.io/badge/Python-3.11%2B-3776AB?logo=python&logoColor=white">
<a href="LICENSE"><img alt="MIT license" src="https://img.shields.io/badge/license-MIT-0A0606"></a>
<a href="https://github.com/devilscrapes/mercadolibre-scraper-python/actions/workflows/ci.yml"><img alt="CI" src="https://github.com/devilscrapes/mercadolibre-scraper-python/actions/workflows/ci.yml/badge.svg"></a>
</p>

This repository is a compact Python client for the **[Mercado Libre Search Results Scraper](https://apify.com/DevilScrapes/mercadolibre-scraper)** on Apify. Hand it a search phrase such as `laptop` and it returns what mercadolibre.com.mx shows for that phrase, one row per organic listing: the item ID, the listing title, who sells it, the star rating, the current price, the pre-markdown price and percentage when there is one, plus the installment and delivery wording from the card. Mercado Libre puts its search pages behind a browser check and has no open search API for this, so the page loading happens on Apify's side under a Mexican residential exit; your script only starts the run and collects the rows. What ends up on disk is `results.json` and `results.csv`, ready for a DataFrame or a nightly price log.

> **Actor:** [Mercado Libre Search Results Scraper](https://apify.com/DevilScrapes/mercadolibre-scraper) · **Publisher:** [Devil Scrapes](https://apify.com/DevilScrapes) · **Also in JavaScript:** [mercadolibre-scraper-javascript](https://github.com/devilscrapes/mercadolibre-scraper-javascript)

## 🔥 What you get

- **Numbers, not strings, for money**: `salePrice` and `listPrice` arrive as numbers in pesos, with `currency` set to `MXN`, so arithmetic works without a cleanup pass.
- **Discounts made explicit**: when a card shows a markdown, the old price lands in `listPrice` and the percentage in `discountPercent`; full-price items carry `null` in both.
- **Organic matches only**: paid placements, which point at click-tracking redirect links, are dropped before a row is built, so ad slots never pollute a price sample.
- **Spanish text kept as written**: `installmentsText` and `shippingText` hold the exact strings from the card (for example a meses-sin-intereses offer), which keeps your own parsing honest.
- **Bounded depth**: 48 results per page, at most 9 pages and 432 rows per run, controlled by `maxItems` and `maxPages`; whichever limit is hit first stops the run.
- **Loud failure instead of a silent empty file**: if Mercado Libre's page check cannot be passed after a few fresh attempts, the run ends as failed with a status message saying so.
- **A search with no hits is not an error**: the run succeeds with zero rows and, because the start event is charged with the first delivered rows, costs nothing.

## 🎯 Use cases

### Watch a category's price level over weeks

Run the same phrase every morning from cron, save each CSV under a dated folder and concatenate them later. With `itemId` as a stable key you can plot a single listing's `salePrice` over time, or take the median per day to see whether the whole category is drifting up or down.

### Size up the sellers in a niche

Group the rows by `seller` and count listings, median price and average markdown. Within a minute you can tell whether a keyword is dominated by two official stores or split across dozens of small shops, which changes how you would enter it.

### Check whether a discount is real

Compare `listPrice` against what you recorded for the same `itemId` last month. A 40 percent badge on a price that was quietly raised the week before tells a different story from a genuine markdown, and the history makes that visible.

### Benchmark payment and delivery terms

Extract the month count from `installmentsText` and flag free delivery from `shippingText`. Analysts studying Mexican e-commerce can then measure how common interest-free plans are at each price band, not just the sticker price.

## ⚡ Quick start

You need an Apify account and an API token from [Console → Settings → Integrations](https://console.apify.com/settings/integrations). Apify's free plan includes monthly usage credit; this Actor's event prices are listed under Pricing below.

```bash
git clone https://github.com/devilscrapes/mercadolibre-scraper-python.git
cd mercadolibre-scraper-python
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
export APIFY_TOKEN=your_token_here
python main.py                      # uses input.json
python main.py my-input.json --out data --max-charge 2
```

The script starts the [Mercado Libre Search Results Scraper](https://apify.com/DevilScrapes/mercadolibre-scraper), waits for it to finish, prints a link to the run in Apify Console and saves every result to `results.json` and `results.csv`. It exits with code 1 and the run's status message if the run fails.

## 🧩 The core call

Strip away the argument parsing and file output and this is all the code you need. The official `apify-client` package starts the Actor, blocks until the run is done and then iterates over every row the run stored.

```python
from apify_client import ApifyClient

client = ApifyClient("YOUR_APIFY_TOKEN")
run = client.actor("DevilScrapes/mercadolibre-scraper").call(run_input={'query': 'laptop', 'tld': 'mx', 'countryCode': 'MX', 'maxItems': 10, 'maxPages': 1, 'proxyConfiguration': {'useApifyProxy': True, 'apifyProxyGroups': ['RESIDENTIAL']}})
for item in client.dataset(run.default_dataset_id).iterate_items():
    print(item)
```

## 📥 Input

`main.py` submits `input.json` as it stands. The bundled file is the same small input that release testing uses: the phrase `laptop`, a single results page and a cap of 10 rows, which keeps a trial run cheap. When you scale up, raise `maxItems` (up to 432) and `maxPages` (up to 9) together, since a page holds 48 listings. Multi-word phrases are fine; spaces are turned into the hyphenated form Mercado Libre uses in its own URLs. The residential proxy group and the `MX` country code are needed for this site, so leave them in place.

```json
{
  "query": "laptop",
  "tld": "mx",
  "countryCode": "MX",
  "maxItems": 10,
  "maxPages": 1,
  "proxyConfiguration": {
    "useApifyProxy": true,
    "apifyProxyGroups": [
      "RESIDENTIAL"
    ]
  }
}
```

| Field | Type | Required | Default | Allowed values |
|---|---|---|---|---|
| `query` | string | yes | `"laptop"` | — |
| `tld` | string | no | `"mx"` | `mx` |
| `countryCode` | string | no | `"MX"` | — |
| `maxItems` | integer | no | `100` | — |
| `maxPages` | integer | no | `3` | — |
| `proxyConfiguration` | object | no | `{"useApifyProxy": true, "apifyProxyGroups": ["RESIDENTIAL"]}` | — |

## 📤 Output

Each row is one organic listing from a search page. Price columns are floats in Mexican pesos and the optional ones become `NaN` in pandas when a card has no markdown or no rating, so filter with `dropna` before averaging. Use the UTC `scrapedAt` value to label snapshots.

| Field | Type | Always present |
|---|---|---|
| `itemId` | string | yes |
| `title` | string | yes |
| `seller` | string | yes |
| `rating` | number or null | no |
| `listPrice` | number or null | no |
| `salePrice` | number | yes |
| `discountPercent` | integer or null | no |
| `installmentsText` | string or null | no |
| `shippingText` | string or null | no |
| `itemUrl` | string | yes |
| `currency` | string | yes |
| `query` | string | yes |
| `page` | integer | yes |
| `scrapedAt` | string | yes |

## 💰 Pricing

| Event | Price (USD) |
|---|---|
| Actor start | $0.2 |
| Result emitted | $0.005 |

Prices as configured on 2026-10-16. The [Actor page](https://apify.com/DevilScrapes/mercadolibre-scraper) always shows the current price. Runs are billed to the Apify account that owns the token. Use `--max-charge` to cap what a single run can cost.

## 🛠️ Working with the data in Python

A first analysis with pandas (`pip install pandas`): it pulls the interest-free month count out of the installment wording, flags free delivery, computes the peso saving on marked-down items, prints a per-seller summary and writes the biggest markdowns to `deals.csv`.

```python
import json

import pandas as pd

df = pd.DataFrame(json.load(open("results.json", encoding="utf-8")))
df["msi_months"] = pd.to_numeric(
    df["installmentsText"].str.extract(r"(\d+)\s+meses sin intereses", expand=False), errors="coerce"
)
df["free_shipping"] = df["shippingText"].str.contains("gratis", case=False, na=False)
df["saving_mxn"] = (df["listPrice"] - df["salePrice"]).where(df["listPrice"].notna())

by_seller = (
    df.groupby("seller")
    .agg(
        listings=("itemId", "size"),
        median_price=("salePrice", "median"),
        avg_discount=("discountPercent", "mean"),
        free_ship_share=("free_shipping", "mean"),
    )
    .sort_values("listings", ascending=False)
)
print(by_seller.round(2).to_string())

deals = df.dropna(subset=["discountPercent"]).sort_values("discountPercent", ascending=False)
cols = ["title", "seller", "salePrice", "saving_mxn", "discountPercent", "msi_months", "free_shipping", "itemUrl"]
deals[cols].to_csv("deals.csv", index=False)
```

## ❓ FAQ

### Is there an official Mercado Libre API I could call instead?

Mercado Libre's developer platform is built around registered applications and seller accounts. This Actor reads the public search pages instead, so the only credential you manage is an Apify token.

### Can I search Argentina, Brazil or Colombia?

Not yet. The `tld` field accepts only `mx`, and `countryCode` must be `MX` to match it. Rows outside Mexico would need separate work per country site.

### Why is my result count lower than `maxItems`?

Either the search ran out of organic matches, `maxPages` stopped it first (48 rows per page), or a card was malformed and skipped with a warning in the log. The run's status message reports how many rows came back across how many pages.

### How do I schedule this and know when it breaks?

Run `python main.py --out snapshots/$(date +%F)` from cron or an Airflow task. A failed Apify run makes the script exit with 1, so the scheduler sees a failure rather than an empty CSV.

### Does it open each product page for specifications or reviews?

No. Everything comes from the search result cards. Detail pages, question threads, full review text and seller storefront pages are outside what this Actor collects.

## 🧯 Troubleshooting

- **`Set APIFY_TOKEN first`**: export the token in the same shell, or put it in a local `.env` file your shell loads. Never commit it.
- **Run finished with `FAILED` or `ABORTED`**: open the printed Console link; the run log says why. Input validation errors name the field.
- **Fewer rows than expected**: a `--max-charge` cap also limits how many results can be charged. Raise the cap or lower the limits in your input.
- **Something looks wrong in the data**: open an issue on the [Actor page](https://apify.com/DevilScrapes/mercadolibre-scraper) so the maintainers see it.

## 🔗 More from Devil Scrapes

- [Mercado Libre Search Results Scraper on Apify Store](https://apify.com/DevilScrapes/mercadolibre-scraper)
- [JavaScript version of this example](https://github.com/devilscrapes/mercadolibre-scraper-javascript)
- [All Devil Scrapes Actors](https://apify.com/DevilScrapes)
- [Apify web scraping cookbook](https://github.com/devilscrapes/apify-web-scraping-cookbook)

Related Actors:

- [AliExpress Products Scraper](https://apify.com/DevilScrapes/aliexpress-products-scraper) — cross-border retail prices for the same product categories.
- [eBay Product & Listings Scraper](https://apify.com/DevilScrapes/ebay-product-listings-scraper) — a second marketplace for price comparison outside Latin America.
- [OLX Brazil Listings Scraper](https://apify.com/DevilScrapes/olx-brazil-scraper) — classified listings from Brazil for a wider LatAm view.

---

MIT licensed example code. The Actor itself runs on [Apify](https://apify.com/DevilScrapes/mercadolibre-scraper) and is maintained by [Devil Scrapes](https://apify.com/DevilScrapes). Not affiliated with Mercado Libre or MercadoLibre, Inc..
