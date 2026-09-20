# Great Lakes

Daily aggregator of Great Lakes aircraft classified listings from
[Barnstormers.com](https://www.barnstormers.com), published as a static page
(`docs/index.html`) meant to be embedded via `<iframe>` on taildraggers.com.

Controller.com was evaluated but dropped: its search results are only reachable
through an internal client-side widget (not a plain URL), which a headless
browser can't drive reliably for an unattended daily job.

## How it works

- `scraper/barnstormers.py` searches Barnstormers.com's
  [Aerobatic - Great Lakes category](https://www.barnstormers.com/category-15869-Aerobatic--Great-Lakes.html)
  for listings, follows pagination, then visits each listing's detail page to pull out
  the price, location, and posted date (falling back to regex heuristics over the
  visible text since the site doesn't expose structured data). The title is derived
  from the listing URL's own SEO slug, since every detail page shares one generic
  `<title>`/`<h1>`.
- Only whole-aircraft-for-sale listings are published. Titles that read as parts,
  accessories, services, or raffles are dropped (see `EXCLUDE_KEYWORDS` in
  `scraper/common.py`) before anything else. Every surviving title is then matched
  against the Great Lakes 2T-1 type-certificate family (2T-1, 2T-1A, 2T-1E, 2T-1LT,
  2T-1MS, optionally with a trailing production-block digit like `2T-1A-2` - WACO
  Classic Aircraft's current-production designation - written with or without
  spaces/hyphens) - see `_MODEL_CODE_RE` in `scraper/barnstormers.py`. Most sellers in
  this category don't repeat a type code at all, so a title with no code (e.g. a bare
  "Biplane", "Great Lakes", or "Waco Great Lakes") still publishes under the generic
  **Sport Trainer** model name (`_MODEL_NAME_RULES`), since the type has had
  essentially one airframe - factory-built or WACO Classic Aircraft-built - across its
  history and Barnstormers already scopes this category to Great Lakes aircraft.
  Every surviving listing's title is rewritten to a canonical **`YEAR Great Lakes
  MODEL`** form when the ad states a model year (e.g. `1929 Great Lakes 2T-1A`), or
  just **`Great Lakes MODEL`**
  when it doesn't - a missing year isn't disqualifying, since plenty of genuine ads
  simply don't state one in the title - regardless of how the original ad was worded,
  so the page reads consistently.
- `main.py` runs the scraper, de-duplicates results, and renders them into
  `docs/index.html` titled **"Other Great Lakes Ads on the Web"**, with one row per
  listing: Title (linked to the original ad), Price, Location, Date Posted, and Site
  Posted On. Links use `rel="noopener noreferrer"` and the page sets a
  `no-referrer` meta policy, so Barnstormers never sees that the click came from
  taildraggers.com.
- `.github/workflows/daily-scrape.yml` runs the whole thing once a day (13:00 UTC),
  commits the regenerated `docs/index.html` if it changed, and can also be triggered
  manually from the Actions tab (`workflow_dispatch`).

## One-time setup: enable GitHub Pages

This repo publishes `docs/index.html` as a plain static file — GitHub Pages just needs
to be pointed at it once:

1. Go to **Settings → Pages** in this repository.
2. Under **Build and deployment → Source**, choose **Deploy from a branch**.
3. Branch: `main`, folder: `/docs`. Save.
4. GitHub will publish the page at `https://taildraggers.github.io/great-lakes/`
   (may take a minute or two the first time).

## Embedding on taildraggers.com

```html
<iframe
  src="https://taildraggers.github.io/great-lakes/"
  title="Other Great Lakes Ads on the Web"
  style="width: 100%; height: 800px; border: 0;"
  loading="lazy">
</iframe>
```

## Running locally

```bash
pip install -r requirements.txt
python main.py
```

This writes/overwrites `docs/index.html`.

## Notes

- If Barnstormers changes its markup or is briefly unreachable, the run logs will
  show a `[warn]`/`[error]` line pointing at what broke rather than failing silently.
- The scraper identifies itself with a browser-like `User-Agent` and adds a short
  delay between requests to be polite to the site.
