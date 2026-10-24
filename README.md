# Web Scraping PHP: Guzzle, DomCrawler and Panther Compared, Plus the Point Where DIY Stops Working

Most PHP scraping questions come down to one decision: which libraries to use for fetching and parsing, and when a hand-rolled script stops being enough. The short answer is Guzzle plus Symfony DomCrawler for anything static, Symfony Panther when JavaScript is involved, and a scraping API like ScraperAPI once your targets fight back with proxies, CAPTCHAs and rate limits.

This guide walks through the working PHP stack first, then the part most tutorials skip — the cost of staying unblocked — and finishes with what ScraperAPI actually costs in practice, including the billing details that don't show up on the pricing headline.

> **Quick picks**
> - Learning or scraping open sites: Guzzle + Symfony DomCrawler. Free, and all you need.
> - JS-rendered pages: Symfony Panther, used only on pages that require it.
> - Production volume against sites that block bots: keep your PHP parsing, route the fetch through ScraperAPI. Two free entry points: a permanent free plan with 1,000 credits (5 concurrent connections), or a 7-day trial with 5,000 credits and no credit card. [👉 Start the ScraperAPI free trial](https://www.scraperapi.com/?fp_ref=coupons)

## The PHP scraping stack, in one table

PHP has four main tools for scraping, each covering a different layer of the problem. Here is what each one does, whether it handles JavaScript, and how hard it is to work with:

| Tool | What it does | JS rendering | Difficulty |
| --- | --- | --- | --- |
| Guzzle / cURL | Fetches HTML and APIs over HTTP | No | Easy |
| Symfony DomCrawler | Parses HTML with CSS/XPath selectors | No | Easy |
| Goutte | Fetch + parse wrapped in one client | No | Easy |
| Symfony Panther | Drives a real browser via WebDriver | Yes | Medium |
| php-webdriver | Low-level Selenium/WebDriver control | Yes | Hard |

A quick note on Goutte, because it trips people up. For years it was the default answer to "how do I scrape in PHP." In 2023 it was effectively absorbed into Symfony and is now a thin wrapper around Symfony's BrowserKit, HttpClient and DomCrawler components. It still installs and works, but its own maintainers now point you at those underlying parts. That's why current guides lean on Guzzle plus DomCrawler directly: same result, actively developed components.

## Building your first PHP scraper with Guzzle and DomCrawler

Install three packages and you have a complete fetch-and-parse setup:

bash
composer require guzzlehttp/guzzle symfony/dom-crawler symfony/css-selector


The `symfony/css-selector` package matters more than it looks. Without it, DomCrawler only accepts XPath expressions; with it, you can use the CSS selectors you already know.

The core loop for almost every PHP scraper is fetch → parse → extract → save. A typical example scrapes a book listing site for titles and prices:

php
$client = new \GuzzleHttp\Client([
    'headers' => [
        'User-Agent' => 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) '
            . 'AppleWebKit/537.36 (KHTML, like Gecko) Chrome/126.0 Safari/537.36',
    ],
    'timeout' => 20,
]);

$html = $client->get('https://books.toscrape.com')->getBody()->getContents();

$crawler = new \Symfony\Component\DomCrawler\Crawler($html);
$books = $crawler->filter('article.product_pod')->each(function ($card) {
    return [
        'title' => $card->filter('h3 a')->attr('title'),
        'price' => trim($card->filter('p.price_color')->text()),
    ];
});

file_put_contents('books.json', json_encode($books, JSON_PRETTY_PRINT | JSON_UNESCAPED_UNICODE));


Three details in that snippet will save you hours of debugging:

1. **The custom User-Agent.** Guzzle's default UA gets blocked by many sites on sight. Send a real browser string.
2. **The try/catch around the request.** Guzzle throws a `RequestException` on 4xx and 5xx responses by default, which is good — it stops you from parsing a 403 error page as if it were data. But you have to catch it yourself in production code.
3. **`filter()` vs `filterXPath()`.** `filter()` takes CSS selectors. When markup gets awkward, `filterXPath()` is there.

For quick one-off scripts without Composer, PHP's built-in `DOMDocument` with a stream context works too — set a User-Agent and timeout in the context options, call `file_get_contents()`, and parse. It handles a simple GET fine; anything involving cookies or POST bodies belongs in Guzzle.

## When to reach for Panther

Guzzle and DomCrawler only see the HTML the server sends. If the data you want is injected by JavaScript — prices loaded via XHR, infinite-scroll product grids — those tools return an empty page and you'll spend an afternoon wondering why your selector matches nothing.

Symfony Panther solves this by driving a real Chrome or Firefox through WebDriver. It runs the page's scripts, waits for the DOM to settle, and then hands you a standard DomCrawler over the rendered result. That means your parsing code barely changes.

The trade-off is real: a browser instance is slower and heavier than an HTTP request. Use it surgically, on the specific URLs that need rendering, and keep Guzzle for everything else. Every step up the stack — Guzzle → Panther → php-webdriver — costs speed and complexity, so only climb when the target forces you to.

## The part tutorials skip: staying unblocked

Here's the reality check. Writing the scraper is the easy 20%. Staying unblocked is the other 80%, and it gets harder every year.

The numbers back this up. Independent benchmarking by Scrapeway in July 2026 found that even the best commercial anti-bot APIs top out around 99% success against protected targets. Proxyway measured some cheaper tools near 33% on defended sites. A hand-rolled PHP script with no proxy rotation does much worse than either.

What does "staying unblocked" actually involve at production volume? Rotating residential proxies, solving CAPTCHAs, mimicking browser fingerprints, retrying failed requests with backoff, distributing requests across geographies. Each of these is its own maintenance burden, and each one breaks silently when a target site changes its defenses.

This is the point where the sensible move is to keep your PHP parsing code exactly as it is and route the fetch through a scraping API. You send a URL plus your API key; the service returns the HTML, having handled proxies, CAPTCHAs and retries upstream. Your Guzzle code changes by a few lines. ScraperAPI is one such service, and [👉 its PHP documentation](https://www.scraperapi.com/documentation/php/?fp_ref=coupons) shows exactly how the request flows work before you commit to anything.

## ScraperAPI plans and pricing

ScraperAPI prices everything in **API credits**, billed monthly, with a 10% discount on annual plans. All plans include the same core feature set: JS rendering, premium residential and mobile IPs, auto-parsing, rotating proxy pools, custom headers and sessions, automatic retries, unlimited bandwidth, and a 99.9% uptime guarantee. The plans differ in credits, concurrency, geo-targeting range and analytics retention.

| Plan | Price (monthly / billed annually) | API credits | Concurrency | Get started |
| --- | --- | --- | --- | --- |
| Free trial | $0 (7 days, no credit card) | 5,000 | 5 | [ Start free trial](https://www.scraperapi.com/?fp_ref=coupons) |
| Hobby | $49 / $44.10 | 100,000 | 20 | [ See Hobby plan pricing](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Startup | $149 / $134.10 | 1,000,000 | 50 | [ See Startup plan pricing](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Business | $299 / $269.10 | 3,000,000 | 100 | [ See Business plan pricing](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Scaling | $475 / $427.50 | 5,000,000 | 200 | [ See Scaling plan pricing](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Professional | $975 / $877.50 | 10,500,000 | 300 | [ See Professional plan pricing](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Advanced | $1,975 / $1,777.50 | 21,500,000 | 500 | [ See Advanced plan pricing](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Enterprise | Custom | 22,000,000+ | 500+ | [ Contact ScraperAPI sales](https://www.scraperapi.com/?fp_ref=coupons) |

One distinction worth knowing before you pick a tier: geo-targeting on Hobby and Startup is limited to US and EU locations, and the analytics dashboard on those tiers only keeps the last 30 days of data. Country-level targeting anywhere in the world starts at Business.

There's also a permanent free plan separate from the trial — sign up and you get 1,000 free API credits with a maximum of 5 concurrent connections, and you can request extra testing credits from support if you need them.

[👉 Compare all plans on the ScraperAPI pricing page](https://www.scraperapi.com/pricing/?fp_ref=coupons)

## Credits are not requests: the real cost math

This is the single most important thing to understand before choosing a plan, and it's the detail most reviews bury: **the credit count on the box is not the number of pages you can scrape.**

A standard page costs 1 credit. But harder targets are priced differently:

- Amazon pages: **5 credits** per request
- Google and Bing searches (including subdomains): **25 credits**
- LinkedIn: **30 credits**
- Sites behind Cloudflare, Datadome or PerimeterX: **+10 credits** per request when the bypass succeeds
- JS rendering: **5–10 credits** per request

Run the math on Hobby: 100,000 credits sounds like 100,000 pages. With JS rendering enabled, it's really 10,000–20,000 requests. If your workload is Google SERPs, it's 4,000. Plan sizing should be based on your actual target mix, not the headline number.

Two tools make this manageable instead of guesswork. The **Domain Cost Estimator** in your dashboard shows the exact cost of any URL before you scrape it. And you can set a `max_cost` per request, which caps what any single scrape can spend — useful insurance when you hit a domain whose cost surprises you.

> A standard page is 1 credit, but Amazon costs 5, Google 25, LinkedIn 30, and bypassing Cloudflare-class protection adds 10. Check your actual targets in the Domain Cost Estimator before committing to a plan — or [👉 test your target mix on the free trial first](https://www.scraperapi.com/?fp_ref=coupons).

## Calling ScraperAPI from PHP

Integration is deliberately thin. ScraperAPI offers five ways to send a request, and all of them work from PHP:

1. **Async Scraper service** at `http://async.scraperapi.com` — submit a job, poll a status endpoint or receive a webhook
2. **Standard API endpoint** at `http://api.scraperapi.com?`
3. **SDKs** (available for some languages)
4. **Proxy port**: `http://scraperapi:APIKEY@proxy-server.scraperapi.com:8001` — swap your Guzzle proxy setting and existing code keeps working
5. **Structured Data service** at `https://api.scraperapi.com/structured/` for pre-parsed JSON from sites like Amazon

The proxy-port option is the one PHP developers tend to appreciate most, because it means your existing Guzzle client doesn't change at all — you route traffic through ScraperAPI's endpoint and it behaves like any other proxy.

The async path is worth knowing for batch jobs. You POST a JSON payload to `https://async.scraperapi.com/jobs` with your API key and target URL, and the service keeps working the URL until it achieves a 100% success rate (when applicable), then delivers the data — either via status polling or a webhook callback. Webhook URLs support basic auth, and the system retries a failing webhook 3 times before cancelling the job, so make sure your endpoint is publicly reachable and can absorb the traffic.

Two limitations belong in your planning, not just your bookmarks:

- **60-second timeout.** Official guidance, regardless of how you invoke the service: set a 60-second timeout in your application for the best success rates, especially on hard-to-scrape domains.
- **2MB per request.** Any response over 2MB won't come back in one piece.

Parameters like `render` (JS rendering), `country_code`, `device_type`, `premium` and `autoparse` go in the `apiParams` object of your request. If you want to try the proxy-port route against your own targets before deciding, [👉 the 7-day trial covers all five access methods](https://www.scraperapi.com/?fp_ref=coupons).

## Where ScraperAPI is the wrong tool

Honesty check, because this affects real purchasing decisions: ScraperAPI performs well on easy targets and modestly on hard ones.

Independent benchmarks put its overall success rate around **62–64%** — just above the industry average of roughly 58–59%, with one benchmark measuring 72.57%. On Cloudflare and DataDome pages specifically, users report outright 500 errors where stronger tools get through. Dedicated unlockers occupy a different tier: Bright Data's Web Unlocker measures around 95–98% success on the same class of protected targets, at a higher price with more onboarding friction.

Speed is a consideration too. Independent benchmarks logged 20–40 seconds per request on Amazon, Idealista and Glassdoor — not our measurements, but the publicly reported figures from those tests. For scheduled batch collection that's acceptable; for anything latency-sensitive it isn't.

One billing quirk to watch: occasionally a CAPTCHA page or error page comes back billed as a successful response. ScraperAPI's official position is that failed requests aren't charged (expected failure rate 1–3%), and support refunds mis-billed CAPTCHA credits when asked — but you need to spot them in your responses first, so spot-check what you're getting.

The practical reading: if your target list is mostly unprotected pages scraped at volume, the simple endpoint and per-credit pricing work in your favor. If most of your targets sit behind serious bot protection, roughly a third of your requests failing — with some failures still billed — erases the price advantage, and a dedicated unlocker is the better fit even though it costs more.

## What happens when you hit your limits

ScraperAPI's behavior at the edge of your plan is more forgiving than most subscriptions, and it's worth understanding before you commit.

Exceed your concurrency limit and the API returns a 429 — the request is turned away rather than queued, and **there's no extra charge** for going over. You can track both request volume and concurrency in the dashboard.

Run out of credits and scraping pauses by default until the next renewal. From there you have options depending on your tier:

- **Auto-upgrade** (opt-in, on Hobby through Professional): your account moves to the next plan automatically when credits run out
- **Pay-as-you-go** (on Scaling, Professional, Advanced and Enterprise): keep scraping past your limit at a fixed per-credit rate, up to a monthly spending cap you set
- **Annual plans** can renew early for a fresh batch of credits, so a heavy month doesn't have to mean waiting for your next billing date

The thing to avoid is silently enabling auto-upgrade on a plan you only intended to test. It's opt-in for a reason — if you're evaluating, leave it off so a heavy week doesn't turn into an upgrade you didn't choose.

## FAQ

**How many requests do I actually get on the Hobby plan?**
It depends entirely on your targets. 100,000 credits equals 100,000 standard pages, 20,000 Amazon pages, 4,000 Google searches, or 10,000–20,000 requests with JS rendering enabled. Check your target domains in the dashboard's Domain Cost Estimator before sizing a plan.

**Does ScraperAPI work with Laravel or Symfony?**
Yes, through any HTTP client — the service is a plain API endpoint or a proxy port. Guzzle integrates directly, and the proxy-port option means existing Guzzle-based code only needs a proxy configuration change.

**Is there a free option for testing?**
Two, technically. A 7-day trial with 5,000 credits and no credit card required, plus a permanent free plan with 1,000 credits and 5 concurrent connections. Support can grant additional testing credits if you ask.

**Can it get past Cloudflare?**
Not reliably. Users report 500 errors on Cloudflare and DataDome targets, and independent benchmarks put overall success at 62–64%. If your workload is mostly protected pages, this is the wrong tool for the job.

**Do failed requests cost credits?**
Officially, no — ScraperAPI says it doesn't charge for failed requests and expects a 1–3% failure rate. The catch is that a CAPTCHA page or error page occasionally comes back marked as a success and gets billed. If you spot one, support will refund those credits; the responsibility for checking your responses is on you.

## Which route should you take?

Match the tool to what you're actually scraping:

- **Learning, prototypes, or open/friendly sites** → Guzzle + DomCrawler, done. Don't pay for anything yet.
- **JavaScript-rendered pages at low volume** → add Panther on the pages that need it. Still free.
- **Production volume on mostly unprotected sites** → ScraperAPI's Hobby tier at $49/month (or $44.10 annually) is the entry point, and the free trial lets you verify your actual success rate and credit burn before spending anything.
- **Mostly Cloudflare/DataDome-protected targets** → ScraperAPI will frustrate you; a dedicated unlocker like Bright Data's Web Unlocker is the better category, at a higher price.

Before you pick a plan, do two things: run your real target list through the Domain Cost Estimator to get your true per-request costs, and spend the 7-day trial measuring your success rate on those exact domains. If the numbers hold up, [👉 get started with ScraperAPI's free trial](https://www.scraperapi.com/?fp_ref=coupons) and keep your parsing code in PHP where it already lives.
