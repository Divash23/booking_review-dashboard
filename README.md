# Booking.com Review Insights Dashboard

An operations dashboard for Azzurro Hotels that collects public Booking.com guest reviews for four Sydney properties, stores them in MongoDB, and turns them into weekly scores, trends and operational topics that non-technical staff can act on.

| Property | Booking.com page |
| --- | --- |
| Olympic Hotel Paddington | https://www.booking.com/hotel/au/olympic-paddington.html |
| Potts Point | https://www.booking.com/hotel/au/venus-potts-point-sydney.html |
| Central Sydney | https://www.booking.com/hotel/au/venus-surry-hills.html |
| Darling Harbour | https://www.booking.com/hotel/au/chateau-de-venus.html |

---

## 1. Setup and installation

**Requirements:** Node.js 20+ and a MongoDB database. A free MongoDB Atlas M0 cluster is enough.

### 1.1 MongoDB Atlas

1. Create a free account at https://www.mongodb.com/cloud/atlas/register.
2. Create an **M0** cluster (AWS, Sydney `ap-southeast-2`).
3. **Database Access** → add a database user with *Read and write to any database*.
4. **Network Access** → add your IP address, or `0.0.0.0/0` for local development.
5. **Connect → Drivers** → copy the connection string.

### 1.2 Backend

```bash
cd server
npm install
npx playwright install chromium
cp .env.example .env
```

Edit `server/.env` and set `Mongodb_Url`. Add the database name `booking-review-dashboard` after `.mongodb.net/`:

```env
base_Port=5050
Mongodb_Url=mongodb+srv://USER:PASSWORD@cluster0.xxxxx.mongodb.net/booking-review-dashboard?retryWrites=true&w=majority
TIMEZONE=Australia/Sydney
REVIEW_LANGUAGE=en
SCRAPER_DELAY_MS=2500
SCRAPER_MAX_PAGES=4
SCRAPER_MAX_PAGES_FULL=40
SCRAPER_MAX_RETRIES=3
SCRAPER_BROWSER_FALLBACK=true
SCRAPE_EVERY_MINUTES=0
DATASET_NAME=Dricz/515k-Hotel-Reviews-In-Europe
```

`.env` is in `.gitignore`. Never commit it.

### 1.3 Frontend

```bash
cd client
npm install
```

The frontend calls `http://localhost:5050/api` by default. To point it somewhere else, create `client/.env` with `VITE_API_URL=https://your-api/api`.

---

## 2. How to run

### 2.1 Load demo data from the open dataset API

```bash
cd server
npm run import:dataset                          # 2,000 reviews, dates moved forward to today
npm run import:dataset -- --limit=5000          # more reviews
npm run import:dataset -- --original-dates      # keep the real 2015–2017 dates
```

This reads real Booking.com guest reviews from the open-source Hugging Face dataset [Dricz/515k-Hotel-Reviews-In-Europe](https://huggingface.co/datasets/Dricz/515k-Hotel-Reviews-In-Europe) (515,738 reviews, Apache-2.0) through `https://datasets-server.huggingface.co/rows`. No key is needed. See section 4.2 for how it is used and labelled.

### 2.2 Collect live reviews (scraper)

```bash
cd server
npm run scrape:full                           # first run: up to 40 pages (≈1,000 reviews) per property
npm run scrape                                # later runs: only new reviews, stops at the first page with nothing new
npm run scrape -- --property=potts-point      # one property
npm run scrape -- --full --pages=10           # custom page limit
```

Each run prints a summary table: status, method (http or browser), pages, reviews found, new, duplicates, skipped. Every run is also saved in the `scraperuns` collection.

Updates can also be started from the dashboard (**Get new reviews**) or scheduled. Set `SCRAPE_EVERY_MINUTES=360` to check every 6 hours while the API is running.

### 2.3 API and dashboard

```bash
cd server && npm run dev      # API on http://localhost:5050
cd client && npm run dev      # dashboard on http://localhost:5173
```

### 2.4 Sample data

```bash
cd server && npm run export   # writes server/sample-data/reviews.json and reviews.csv
```

### 2.5 Fallback: CSV import

If Booking.com blocks automated access, reviews can be loaded from a CSV, for example one exported from the Booking.com Extranet or collected manually:

```bash
npm run import:csv -- ./my-reviews.csv
```

The columns it accepts are `property, review_date, rating, reviewer_name, reviewer_country, title, liked, disliked, room_type, stay_date, traveller_type, review_id`. Duplicates are skipped the same way as in the scraper.

---

## 3. Architecture

```text
 Booking.com public review pages        Hugging Face open dataset API        CSV file
            │ reviewlist.html?pagename=…                │ /rows?dataset=…&offset=…          │
            ▼                                           ▼                                   ▼
 ┌─────────────────────── server (Node.js + Express) ───────────────────────┐
 │ Scraper/fetchPage.js     HTTP request → retries + backoff → Playwright    │
 │ Scraper/parseReviews.js  cheerio: 2 page layouts → clean review objects   │
 │ Scraper/saveReviews.js   analyse + upsert by ExternalId (no duplicates)   │
 │ Scraper/scrapeProperty.js paging, incremental stop, ScrapeRun log         │
 │ Scraper/importDataset.js open dataset → same save path, labelled "Demo"   │
 │ Utils/analyzeReview.js   sentiment + 8 operational topics                 │
 │ Controller/*             summary, weekly, properties, topics, insights    │
 │ Routes/*                 /api/property  /api/review  /api/dashboard       │
 │                          /api/scrape                                      │
 └──────────────────────────────────┬────────────────────────────────────────┘
                                    │ Mongoose
                           MongoDB Atlas: properties · reviews · scraperuns
                                    │ REST (JSON)
 ┌──────────────── client (React + Vite + Tailwind + Recharts) ──────────────┐
 │ page/Dashboard.jsx  filters + data loading                                │
 │ Component/*         statistics, ratings, properties, trends, insights,    │
 │                     topics, review feed, data status                      │
 └───────────────────────────────────────────────────────────────────────────┘
```

**Folders**

```text
server/
  Config/       db.js, settings.js (env), properties.js (the 4 hotels)
  Model/        Property, Review, ScrapeRun
  Scraper/      fetchPage, parseReviews, saveReviews, scrapeProperty, runScraper, importDataset
  Utils/        analyzeReview, buildFilter, dates, ensureProperties
  Controller/   Property, Review, Dashboard, Scrape
  Routes/       Property, Review, Dashboard, Scrape
  index.js      Express app
  scrape.js     CLI scraper
  importDataset.js  CLI open-dataset import
  importCsv.js  CSV fallback
  exportSample.js
client/src/
  Api/api.js    all API calls
  page/         Dashboard.jsx
  Component/    UI components
  lib/utils.js  formatting helpers
```

---

## 4. Review collection method

### 4.1 Live Booking.com scraper

Booking.com has no public reviews API, so the scraper reads the same public review list that the hotel page loads when a visitor opens “Guest reviews”:

```
https://www.booking.com/reviewlist.html?cc1=au&pagename=olympic-paddington&type=total&sort=f_recent_desc&rows=25&offset=0&lang=en-gb&r_lang=en
```

1. **Fetch.** A normal HTTPS request with a browser user agent, a 20-second timeout, and up to 3 retries with exponential backoff and jitter. Retries happen on timeouts, HTTP 429 (honouring `Retry-After`) and 5xx. A 404 stops at once, because it means the page name is wrong.
2. **Browser fallback.** If Booking.com answers with a bot check (HTTP 202/403 or a challenge page), the run switches to a headless Chromium browser via Playwright (open source, Apache-2.0) for the rest of that run.
3. **Parse.** `cheerio` (open source, MIT) reads each review. Two layouts are supported: the classic `c-review-block` list and the newer `data-testid="review-card"` layout. Fields: reviewer name, country, score (1–10), title, liked text, disliked text, review date, stay month, nights, room type and traveller type.
4. **Validate.** A review without a readable score or date is skipped and counted, not guessed. Placeholder text such as “There are no comments available” is removed.
5. **De-duplicate.** Each review gets an `ExternalId`: Booking.com's own review id when the page has one, otherwise a SHA-1 fingerprint of property + reviewer + date + score + text. MongoDB has a unique index on it, and saves use upserts, so re-running never creates duplicates.
6. **Incremental updates.** Reviews are requested newest first. A normal run stops at the first page that has no new reviews, which is usually one request per property. `scrape:full` back-fills history.
7. **Detecting page changes.** If the first page returns HTML but no reviews can be found, the run is marked `failed` and the HTML is saved to `server/logs/snapshots/`. That shows what changed and stops the dashboard showing stale data as if it were fresh.
8. **Run log.** Every property run is stored in `scraperuns` with status (`success`, `partial`, `failed`, `blocked`), method, counts and errors. The dashboard shows this under **Data Updates**.
9. **Politeness.** One property at a time, 2.5–3.5 seconds between pages, and only one run at once (a second start returns HTTP 409).

### 4.2 Open dataset (demo data)

The live scraper could not be verified from the build environment, and Booking.com may block automated access. So the project also loads reviews from an open-source dataset of real Booking.com reviews: [Dricz/515k-Hotel-Reviews-In-Europe](https://huggingface.co/datasets/Dricz/515k-Hotel-Reviews-In-Europe), read through the Hugging Face Datasets API.

- **Same shape as Booking.com.** The dataset has a score out of 10, separate liked and disliked text, reviewer nationality, and tags (room, nights, traveller type, trip type). It goes through the same analysis and save path as scraped reviews.
- **Spread across hotels.** Rows are read in pages of 100, spread evenly across the 515,738 reviews, so many different hotels are included. Each source hotel is always mapped to the same one of the four properties.
- **Dates are moved forward.** The data ends on 3 Aug 2017, so by default all dates are shifted by the same amount to end today. This keeps weekly comparisons meaningful. `--original-dates` turns this off.
- **Labelled as demo data.** Every review keeps `Source = "Demo dataset (515K Hotel Reviews Europe)"` and its real `SourceHotel`. The dashboard shows an “Includes demo data” badge and the source hotel on each review card. **These reviews are not reviews of the Azzurro properties.**
- **Reliability.** 1 second between requests, retries with backoff, `Retry-After` honoured on HTTP 429, partial imports kept and logged, and re-running never creates duplicates (`ExternalId = dataset:<row>`).

---

## 5. Review insights methodology

**Sentiment.** The score is `0.7 × rating component + 0.3 × text component`:
- **Rating component:** the Booking.com score, scaled so that 6.5/10 is neutral.
- **Text component:** the open-source [`sentiment`](https://www.npmjs.com/package/sentiment) library (AFINN word list) applied to the liked, disliked and title text. Any real “disliked” text counts as a complaint.

The result is ≥ 0.2 Positive, ≤ −0.2 Negative, and Neutral otherwise. In practice, 8+ with no complaints is positive, 5 or below is negative, and 6–7 depends on the text.

**Topics.** Eight operational topics: Cleanliness, Check-in, Staff, Noise, Facilities, Location, Room condition and Value for money. Each is matched with a keyword list on word boundaries. Booking.com splits each review into *liked* and *disliked*, which is the main design advantage:
- topics found in the **disliked** text become `IssueTopics` (complaints)
- topics found in the **liked** text become `PraiseTopics`

So “quiet room” counts as praise for Noise, not a noise complaint.

**Insights generated** (`/api/dashboard/insights`):
- this week's average vs last week (weeks run Monday to Sunday, Australia/Sydney)
- “X% of negative reviews this week mentioned cleanliness-related issues (n of N)”. If this week has fewer than 5 negative reviews, it uses the last 30 days instead and says so.
- topics whose complaint share rose by 10 or more points vs last week
- the most common complaint per property
- the best and worst property in the selected period

---

## 6. Known limitations and assumptions

- **Terms of use.** Automated collection may conflict with Booking.com's terms. The scraper is deliberately slow and low-volume and only reads public pages. For production, the more reliable options are the Booking.com Connectivity/partner APIs (available to property owners through a connectivity provider) or the Extranet. The CSV import exists for that route.
- **Markup changes.** Booking.com changes its HTML without notice. Selectors live in one file (`Scraper/parseReviews.js`), two layouts are supported, and failures are detected and logged. Some maintenance will still be needed.
- **Bot protection.** Booking.com can block requests. The browser fallback helps, but no captcha solving or proxy rotation is done, on purpose. A blocked run is shown as “Blocked by Booking.com” instead of failing silently.
- **Language.** By default only English reviews are collected (`REVIEW_LANGUAGE=en`), because topic keywords are English. Set it empty to collect all languages; topics will then be missed for non-English text.
- **Keyword topics.** These are transparent and explainable, but miss synonyms and sarcasm, and can catch words used in another sense (“room for improvement”). An LLM classifier could be added later in `Utils/analyzeReview.js` without changing the rest of the app.
- **Review dates.** Booking.com shows the review date by day. Stay dates are only a month (“March 2026”) and are stored as text.
- **Edited or deleted reviews.** An edited review gets a new fingerprint when Booking.com gives no id. A review deleted on Booking.com stays in the database (`LastSeenAt` shows when it was last seen).
- **Small weeks.** With few reviews per week, percentages swing a lot. Counts are always shown next to percentages (“4 of 10”), and the negative-review breakdown uses the last 30 days when the current week has fewer than 5 negative reviews.
- **Demo dataset.** The open dataset is real Booking.com data, but from European hotels in 2015–2017. It is mapped onto the four properties with shifted dates for demonstration only, and labelled as such everywhere. Operational decisions need the live scraper or a CSV export from the Extranet.
- **Not live-tested from the build environment.** The scraper was tested end to end against local copies of the review-list format (paging, retries, de-duplication, incremental stop, layout-change detection). The live selectors must be checked on the first real run: run `npm run scrape -- --property=potts-point --pages=1` and look at the summary.

---

## 7. API reference

Base URL `http://localhost:5050/api`. These filters work on every review and dashboard endpoint: `property` (id), `startDate`, `endDate` (YYYY-MM-DD, Sydney time), `sentiment`, `topic`, `issue`, `search`.

| Method | Endpoint | Purpose |
| --- | --- | --- |
| GET | `/property/all-property` | The four properties |
| GET | `/property/single-property/:pid` | One property + review count |
| GET | `/review/all-review?page=&limit=&sort=newest\|lowest\|highest` | Review feed |
| GET | `/review/single-review/:rid` | One review |
| GET | `/dashboard/summary` | Totals, sentiment split, this week vs last week |
| GET | `/dashboard/weekly-rating?weeks=8` | Weekly average and positive/negative counts |
| GET | `/dashboard/property-performance` | Per-property score, weekly change, top issue |
| GET | `/dashboard/sentiment` | Positive / neutral / negative counts |
| GET | `/dashboard/topics` | Mentions, praise and complaints per topic |
| GET | `/dashboard/insights` | Plain-English insights |
| POST | `/scrape/run` | Start an incremental live update (409 if one is running) |
| POST | `/scrape/import-dataset` | Import from the open dataset, body `{ "limit": 2000 }` (409 if one is running) |
| GET | `/scrape/status` | Last run per property |
| GET | `/scrape/history` | Last 30 runs |

Errors are returned as `{ "message": "…" }` with status 400, 404, 409 or 500.
