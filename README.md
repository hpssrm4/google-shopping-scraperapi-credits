# Scrape Google Shopping: What the JSON Returns, Which Filters Work, and How 25 Credits per Request Decides Your Plan

Google answers an ordinary HTTP request to its Shopping results with a "JavaScript required" shell page or an "unusual traffic" check, and neither one contains a product card. Your HTML parser gets nothing to parse, which is why `requests.get()` tutorials that stop at the download step fall over on the first run. The work below covers what ScraperAPI's Google Shopping endpoint actually returns, which parameters change the results, five ways a run quietly wastes credits, and the arithmetic that tells you which plan your volume needs.

> **Quick pick for most readers:** the free tier. It gives 1,000 credits and 5 concurrent connections, and a Google request costs 25 credits, so that's 40 Shopping page requests before you pay anything. At 40 to 70 products per page, that's roughly 1,600 to 2,800 product rows, enough to prove the pipeline and check whether Google's cards carry the fields you need. 👉 [Start on the free plan and see current tier prices](https://www.scraperapi.com/pricing?fp_ref=coupons)

## The plan ladder, and the number that decides your bill

Every request to a Google domain costs 25 credits on ScraperAPI, on any tier. A standard page costs 1 credit, Amazon 5, LinkedIn 30, and sites sitting behind Cloudflare, Datadome or PerimeterX add 10 when ScraperAPI bypasses them. That multiplier divides your monthly allowance by 25 before you see a single product, so credits matter more here than the sticker price of the plan.

| Plan | Where it sits | Keeping a run alive past the cap | Get it |
| --- | --- | --- | --- |
| Free | 1,000 credits, 5 concurrent connections | Scraping pauses until credits renew | [Get started](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Hobby | First paid tier | Opt-in auto-upgrade to the next tier when credits run out | [Choose Hobby](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Startup | Above Hobby | Auto-upgrade | [Choose Startup](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Business | Above Startup | Auto-upgrade | [Choose Business](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Scaling | First tier with pay-as-you-go | Auto-upgrade plus PAYG at a fixed per-credit rate, up to a monthly cap you set | [Choose Scaling](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Professional | Above Scaling | Auto-upgrade plus PAYG | [Choose Professional](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Advanced | PAYG only | PAYG at a fixed per-credit rate | [Choose Advanced](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Enterprise | Custom | Custom pricing across your usage; Ultra Premium requests carry their own minimum rate | [Talk to sales](https://www.scraperapi.com/pricing?fp_ref=coupons) |

Two behaviours affect a scheduled job more than the tier name. Exceeding your concurrency returns a 429 and the request is turned away rather than queued, with no extra charge for trying. Running out of credits pauses scraping by default unless auto-upgrade or PAYG is switched on, and annual plans can renew early for a fresh batch.

ScraperAPI sets tier prices on the pricing page and they differ between monthly and annual billing, so the table above tracks the mechanics that actually differ between tiers instead of quoting figures that move.

## What the Google Shopping endpoint hands back

You call it through Structured Data Endpoints, picking Google Shopping from the endpoint list, and the response arrives as JSON by default or as CSV when you pass `output_format=csv`. Each item in `shopping_results` carries the fields below.

| Field | Type | What it's for |
| --- | --- | --- |
| `position` | integer | Rank inside that one results page, nothing more |
| `title` | string | Product name as printed on the card |
| `price` | string | Formatted price, for example `$199.00`, display only |
| `extracted_price` | float | The number to compare, sort and threshold on |
| `source` | string | Retailer shown on the card |
| `delivery_options` | string | Shipping text, when Google prints it |

Two things trip up importers. `price` and `extracted_price` are the same figure in two formats, so filtering on the string version gives you `$1,099.00` sorting below `$199.00`. And `position` reshuffles between runs, which makes it a poor record key. Where your endpoint exposes a Google product identifier, key on that. Where it doesn't, title plus source is the fallback, and it needs fuzzy matching because Google rewords titles.

One card equals one offer. ScraperAPI reads the search results page, not the product detail panel, so a product sold by six stores shows up as one card and may say "& more" without naming the other sellers.

## Parameters that decide which market and page you get

| Parameter | Effect |
| --- | --- |
| `query` | Search terms, the one required field |
| `country_code` | Two-letter market code for the Shopping results |
| `tld` | Google domain: `com`, `co.uk`, `ca`, `de`, `fr`, `it`, `co.jp`, `in`, `com.au`, `com.br`, `nl`, `se`, `pl`, `com.sg`, `com.mx`, `ae`, `com.tr`, `sa`, `cn` |
| `uule` | Encoded city for local results, needs a UULE encoder to generate |
| `num` | Results requested per page |
| `hl` / `gl` | Interface language and country boost |
| `start` | Result offset, the paging control |
| `output_format` | `json` (default) or `csv` |

`country_code` and `tld` are separate controls, and scraping `google.com` from Canada needs both set. Prices, merchants and availability change with the market, so a query run against `us`, `gb` and `de` returns three different datasets rather than three translations of one.

How many cards actually land on a page is Google's decision. Third-party tests put it at roughly 40 to 70 depending on how broad the query is. `num` is a request, not a guarantee, and setting it high enough to exceed one page triggers paging.

## Price, shipping and sale filters live somewhere else

ScraperAPI's documented Shopping parameters cover market, language, city, page size and offset. Price ceilings, free-shipping-only and on-sale-only aren't in that list, so you filter after the response lands: `extracted_price` for the price band, the delivery text for shipping, and your own comparison against stored rows for discounts.

Dedicated Shopping APIs put those filters on the request. SerpApi takes `min_price`, `max_price`, `free_shipping`, `on_sale`, `small_business`, `sort_by` (1 for low to high, 2 for high to low) and `device`. Apify's actors add condition (new, used, refurbished), store names, free returns and sort-by-rating across 29 country markets. Server-side filtering saves storage and post-processing when your target is narrow, though the request still costs the same 25 credits per Google page either way.

## Five ways a run burns credits and returns nothing useful

**No proxy means no products.** Google will not serve an unproxied request, and both of its refusal pages are empty of listings. If you're hand-rolling calls to `google.com/search?tbm=shop`, this is the wall you hit first, and it's the reason a managed endpoint exists.

**Images come back null more often than any other field.** Google lazy-loads thumbnails and publishes only some URLs in the page source, with the rest arriving in a follow-up request that a plain HTTP scraper never makes. Top-of-page products are consistently among the missing. ScraperAPI's documented Shopping fields don't include an image URL at all, so if you need thumbnails, plan a second source for them.

**Merchant, rating, review count and shipping can be null.** Google doesn't print all of them on every card, so your importer has to accept nulls in those columns from day one. A schema that requires a rating will reject perfectly valid rows.

**Two runs minutes apart differ.** Location, time of day and session all shift the ordering and the prices. Store a timestamp with every row and compare by product ID, because a "price change" between position 3 and position 5 is usually two different products.

**`wasPrice` and `discount` only exist when Google badges a reduction.** They're sourced from Google's "Was" and "Usually" labels, so an absent previous price says nothing about whether the price moved. Real price history is something you accumulate yourself, one snapshot at a time.

## What a real job costs in credits

At 40 to 70 products per page, 1,000 products is 15 to 25 page requests, or 375 to 625 credits. The free tier's 1,000 credits covers that comfortably.

A recurring job scales fast. Ten keywords at two pages each is 20 requests a day, 500 credits, and around 15,000 credits a month. Running the same query across five markets at two pages is 10 requests, or 250 credits per sweep, so country coverage multiplies the bill linearly.

Three tools keep the estimate honest before you commit. The Summary panel in the API Playground shows the cost of a job before you run it. The Domain Cost Estimator in your dashboard returns the exact credit cost for any URL. And the `max_cost` parameter caps what a single request is allowed to spend.

For batches, switch the Playground to Async and paste your queries as full structured endpoint URLs, one per line, then poll the `statusUrl` for each job until the status reads `finished`. That's what makes a 6am crontab running ten keywords across three markets practical. 👉 [Run your first 40 Shopping requests on the free tier](https://www.scraperapi.com/pricing?fp_ref=coupons)

## Compliance: what stays in bounds

Shopping cards are public, sit behind no login and carry no personal data, and *hiQ Labs v. LinkedIn* is the usual precedent cited for scraping material anyone can see. That's the boundary, and it doesn't move: product listings yes, account data no. Google's terms of service and your local law still apply to how you use the results, and that responsibility sits with you rather than the API.

## ScraperAPI against a dedicated Shopping API

SerpApi's Google Shopping endpoint documents a wider field set than ScraperAPI's, including `product_id`, `rating`, `reviews`, `link`, `product_link`, `thumbnail` and `extensions`, and it takes price, shipping, sale and small-business filters directly. Apify's two Google Shopping actors publish per-item field lists with brand, condition, returns and multi-store flags, and bill per product saved to the dataset rather than per request.

Both are built for one job. ScraperAPI's advantage is that the same credit pool covers every other site you scrape, with one key, one dashboard and one billing line, and its Shopping endpoint still returns structured JSON or CSV without you maintaining a parser. ScraperAPI's documented Shopping fields are leaner, and the filters you'd want on the request are applied after it.

If Google Shopping is the entire product and you need every field plus server-side filtering, a dedicated API earns its keep. If Shopping is one source among a dozen and you'd rather not run two vendors, keeping it inside the ScraperAPI allowance is less work.

## FAQ

**Why is my dataset empty even though the job finished?** You're almost certainly hitting Google without a proxy. The refusal pages load fine, look like HTML, and contain zero products, so a scraper that only checks the HTTP status code reports success on an empty result. Check the response body for "JavaScript required" before you debug anything else.

**Can I get product images?** ScraperAPI's documented Shopping fields don't include one. On endpoints that do return images, expect frequent nulls from lazy loading, and expect top-of-page products to be the most affected.

**Why did the same query return different products an hour later?** Google personalises Shopping by location, time and session. Compare runs by product ID and keep a timestamp on each row; ranking position is not a stable identifier.

**How many products fit on one page?** Roughly 40 to 70 depending on the query. To go past that, page with `start` and budget 25 credits for each additional page.

**Can I pull historical prices?** No endpoint returns them. Google shows a "Was" price only for products it has badged as reduced, which is not the same as a real price history. Store daily snapshots and build the history yourself.

**Can I filter to on-sale or free-shipping products before the request?** Not with ScraperAPI's documented parameters. Filter on `extracted_price` and the delivery text after the response, or use a dedicated Shopping API that accepts those filters.

## Which tier fits your workload

One-off category research, or a first test of whether Google's card data covers what you need: stay on the free tier. Forty page requests is 1,600 to 2,800 rows, and that answers most feasibility questions.

Daily monitoring of a handful of keywords in one market: work out your monthly credits first. Ten keywords at two pages a day lands near 15,000 credits, and you want a tier whose allowance clears that with room for the odd wider sweep. Check the concurrency limit on the tier you pick, since parallel keywords queue against it and a 429 just turns the request away.

Multi-market price comparison: credits scale with countries, so five markets multiplies your monthly figure by five. That's the point where pay-as-you-go on Scaling or above is easier to reason about than jumping two tiers.

Before you pay, confirm three things: the credit allowance on the tier, the exact cost the Domain Cost Estimator returns for a real Shopping URL, and whether your tier offers auto-upgrade or PAYG when a scheduled run overshoots. Those three numbers decide your actual bill; the headline price doesn't. 👉 [Compare ScraperAPI plans and pick your credit allowance](https://www.scraperapi.com/pricing?fp_ref=coupons)
