# web scraping node js cheerio: What Cheerio Can and Can't Do, How to Handle JS-Rendered Pages, and What a Scraper API Actually Costs

Most tutorials on web scraping node js cheerio stop at the same place: install Cheerio, parse some HTML, done. What they usually skip is the part where your real project breaks — the target site renders everything with JavaScript, or Cloudflare starts serving you challenge pages at request 500. This guide covers both halves: how to build the fetch + Cheerio stack properly, how to tell quickly whether Cheerio is even the right tool for your target, and what it costs to hand the hard part (rendering, proxies, CAPTCHAs) to a scraper API like ScraperAPI — with the full plan table and the credit rules that most reviews leave out.

> **Quick take:** if your targets are static HTML pages, `fetch` + Cheerio is free and all you need. If you're hitting JS-rendered or well-protected sites at any scale, pairing Cheerio with a scraper API is the pragmatic move. ScraperAPI starts at **$49/month for 100,000 credits** (Hobby), with a free plan that gives you 1,000 credits and a 7-day trial of 5,000 credits — no credit card required. 👉 [Check the current ScraperAPI plans and start the free trial](https://www.scraperapi.com/signup?fp_ref=coupons)

## What Cheerio actually does in a Node.js scraping stack

Cheerio is a parser, not a browser. It takes an HTML string and lets you query it with jQuery-style selectors — `$(".product .price").text()` — but it never downloads anything, never executes JavaScript, and never manages a browser session. Because it doesn't launch a browser, it's fast: it can parse thousands of pages per second on modest hardware.

That's why the standard pipeline has two steps and two tools:

1. An HTTP client (the built-in `fetch` on Node 18+, or Axios) downloads the raw HTML.
2. Cheerio turns that HTML into a queryable document.

It's the same division of labour as `requests` + BeautifulSoup in Python: one tool fetches, the other parses. Since Node 18 ships a global `fetch`, a basic scraper needs exactly one dependency:

bash
npm install cheerio


Set `"type": "module"` in your `package.json` (or name the file `.mjs`) and you can use top-level `await` with ES module syntax.

A minimal working loop, against the books.toscrape.com practice sandbox:

js
import * as cheerio from "cheerio";

const url = "https://books.toscrape.com/catalogue/page-1.html";
const res = await fetch(url, {
  headers: { "User-Agent": "Mozilla/5.0 (compatible; BookScraper/1.0)" },
});
const html = await res.text();
const $ = cheerio.load(html);

const books = [];
$("article.product_pod").each((_, el) => {
  books.push({
    title: $("h3 a", el).attr("title"),
    price: $(".price_color", el).text(),
  });
});


Three details that matter more than they look. First, set a browser-like User-Agent on every real scrape — many sites block default or empty agent strings. Second, use `$(el).find(...)` (or scoped selectors) to keep sub-queries inside the current element, or you'll pull fields from the wrong product. Third, convert relative links with `new URL(href, url)` instead of string concatenation; nested pages will silently produce wrong base paths otherwise.

Pagination works the same way: read the "next" link with something like `$("li.next a").attr("href")`, fetch, repeat until the selector comes up empty, and put a short delay between requests so you stay inside the site's rate limits. Cheerio holds no state between requests — the loop is yours to write.

## The two problems Cheerio can't solve on its own

Cheerio fails in two specific ways, and it's worth knowing which one is biting you before you reach for heavier tools.

**JavaScript-rendered pages return an empty shell.** If the site's HTML looks like `<div id="root"></div>` plus a pile of script tags, the content is built in the browser and Cheerio's selectors will match nothing. There's a 10-second diagnostic for this: `console.log(html)` on the raw response before it goes into `cheerio.load()`. If your target data is in that string, Cheerio works. If it isn't, no amount of selector tweaking will help.

When the data isn't there, you have two options, in order of preference:

1. **Find the hidden API.** Open DevTools, watch the Network tab filtered to Fetch/XHR, and find the JSON endpoint the page's JavaScript is calling. Hit it directly with Axios and you skip rendering entirely — this is usually the best path, and you can still parse any embedded HTML fragments with Cheerio.
2. **Render with a real browser.** Playwright or Puppeteer runs the page's scripts, and you feed the rendered HTML back into Cheerio for parsing. This works, but it costs you speed and memory — Playwright bundles its own matching Chromium build (current versions ship around 148.0.7778.96), and `npx playwright install chromium` is the slowest part of any setup.

The difference is not subtle. In one documented test run on Node v22.18.0, fetch + Cheerio pulled all 1,000 books across 50 pages of a static sandbox, but on a JavaScript-rendered quotes page the same fetch found **0** quotes and Playwright found 10. Same code style, completely different outcome.

**Blocking kicks in at scale.** A default Axios request has a fingerprint that protection systems recognize: a generic user-agent, no browser-style headers, a datacenter IP, and a TLS signature that doesn't look like Chrome. On a small project against a friendly site, none of this matters. Once you're doing thousands of requests, or the target sits behind Cloudflare, DataDome, or Akamai, Axios can't solve the challenge pages — it doesn't run a browser, so the JavaScript challenge never completes. Rotating proxies and realistic headers help on medium targets, but strong protection needs a real browser environment or a service that maintains one.

## Where ScraperAPI fits in the Cheerio pipeline

ScraperAPI takes over the fetch step, not the parsing step. You send your target URL to their endpoint with your API key, their infrastructure handles proxies, browser rendering, CAPTCHAs and IP rotation, and you get back HTML that you then load into Cheerio exactly as before. Your parsing code doesn't change; only the source of the HTML does.

That split has a practical benefit: keeping the parsing logic in Cheerio means switching scraping providers later touches one line of your code, not your whole extraction layer. Independent benchmarkers have used exactly this architecture — keep Cheerio for parsing, swap only the fetch layer — when comparing providers.

You control cost and behaviour with query parameters. The ones that matter for a Node.js + Cheerio workflow:

- `render=true` — runs the page's JavaScript before returning HTML. This is the parameter that replaces a Playwright dependency for most JS-rendered targets. It costs 10 extra credits per request.
- `premium=true` — routes the request through premium residential proxies, 10 extra credits.
- `country_code`, `session_number`, `device_type`, `wait_for_selector`, `autoparse` — no extra cost. Note that `wait_for_selector` lets you hold the response until a specific element appears, which pairs naturally with a Cheerio selector you already know works.
- `max_cost` — caps how many credits a single request may consume.

So the decision rule is straightforward: if your pages render in the browser, you can either run Playwright yourself (free, but you manage the browser and the proxying) or add `render=true` and let ScraperAPI do it. If you want to see what the free tier can do against your actual targets before committing, 👉 [grab the 5,000-credit trial and test your own URLs](https://www.scraperapi.com/signup?fp_ref=coupons) — the onboarding includes an API Playground where you can run a request and see exactly what it consumed. And if the free allocation runs dry during testing, support can extend additional test credits on request, so you don't have to pay just to finish evaluating.

## ScraperAPI pricing: every plan, and what credits actually cost

Here's the full plan list as shown on ScraperAPI's pricing page. Annual billing takes 10% off every tier:

| Plan | Monthly price | Annual (−10%) | Credits / month | Concurrent threads | Purchase |
| --- | --- | --- | --- | --- | --- |
| Hobby | $49 | $44.10 | 100,000 | 20 | [Start Hobby](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Startup | $149 | $134.10 | 1,000,000 | 50 | [Start Startup](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Business | $299 | $269.10 | 3,000,000 | 100 | [Start Business](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Scaling (most popular) | $475 | $427.50 | 5,000,000 | 200 | [Start Scaling](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Professional | $975 | $877.50 | 10,500,000 | 300 | [Start Professional](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Advanced | $1,975 | $1,777.50 | 21,500,000 | 500 | [Start Advanced](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Enterprise | Custom | Custom | 22,000,000+ | 500+ | [Contact sales](https://www.scraperapi.com/pricing?fp_ref=coupons) |

Every paid plan includes JS rendering, premium proxies, automatic CAPTCHA handling, JSON auto-parsing, rotating proxy pools, custom headers and sessions, automatic retries, unlimited bandwidth and a 99.9% uptime guarantee. There's also a permanent free plan: 1,000 credits on signup with a maximum of 5 concurrent connections, plus the 7-day trial with 5,000 credits that requires no card.

The two differences between tiers that actually affect most buyers: **geotargeting** (Hobby and Startup are limited to US & EU locations; Business and up get country-level targeting) and **analytics retention** (30 days on the lower tiers, unlimited from Business up). Concurrent threads rise with every tier, and exceeding your thread limit returns a 429 — the request is rejected rather than queued, and you're not charged for it.

Now the part that pricing-page screenshots never show: **a credit is not always one request.** A standard page costs 1 credit, but domain-specific pricing applies:

| Target type | Credits per request |
| --- | --- |
| Normal pages | 1 |
| Amazon, Walmart, eBay | 5 |
| Google, Bing (all subdomains) | 25 |
| LinkedIn | 30 |
| Cloudflare / DataDome / PerimeterX / Akamai bypass | +10 per request |

On top of that, request parameters stack: `render=true` adds 10, `premium=true` adds 10, `screenshot=true` adds 10, and `ultra_premium=true` (paid plans only) adds 30. Combining `premium` with `render` costs 25; `ultra_premium` with `render` costs 75. Parameters like `country_code`, `session_number` and `autoparse` stay free.

Run the arithmetic before you pick a plan, because the mismatch is easy to miss. Suppose you're scraping Amazon product pages with JS rendering: that's 5 (domain) + 10 (render) = 15 credits per page. The Hobby plan's 100,000 credits therefore buys about **6,600 Amazon pages per month**, not 100,000 requests. The same math for plain static pages behind no protection: 100,000 pages, which is why Hobby is genuinely enough for many Cheerio-based projects. If you're unsure what a specific target will cost, use the Domain Cost Estimator in the dashboard before committing, or call the cost endpoint with your API key and any parameters you plan to use — it returns the exact per-request price.

Three billing details worth knowing before you subscribe:

- Only successful requests are billed — 200 and 404 responses. But a request you cancel before 70 seconds also gets billed, so don't fire and abort.
- When credits run out, scraping pauses until renewal by default. On Hobby through Professional you can opt into auto-upgrade, which bumps you to the next plan automatically. On Scaling and above there's pay-as-you-go at a fixed per-credit rate, capped at a monthly spend limit you set.
- Each response includes a `sa-credit-cost` header showing what that request actually consumed — useful for spotting cost surprises in your own logs.

If that cost model makes sense for your targets, 👉 [compare the plan tiers on ScraperAPI's pricing page](https://www.scraperapi.com/pricing?fp_ref=coupons) and do the per-request math for your specific domains before choosing between monthly and annual billing.

## How ScraperAPI compares to the alternatives

ScraperAPI is the budget-friendly option for easy-to-moderate targets, but it's not the strongest on heavily protected pages, and independent testing shows it. In Scrapeway's July 2026 benchmark, ScraperAPI's real success rate on hard-protected pages landed around **62–64%**, while Scrapfly ranked first overall at roughly 99% with JavaScript rendering, and Bright Data posted the highest anti-bypass success of the major vendors — at higher prices and with a KYC verification process.

Reading those numbers honestly: they're ceilings measured against difficult targets, and your own mileage depends entirely on which sites you're scraping. For static and lightly protected pages — the majority of Cheerio use cases — the 62–64% figure is largely irrelevant and ScraperAPI's price per credit is the more relevant axis. If your whole project is scraping a handful of heavily fortified sites at high volume, that's the scenario where a premium-priced provider earns its cost, and ScraperAPI is probably the wrong pick. Whatever you lean toward, run your real targets through a trial before committing money to any provider.

## FAQ

**Do I still need Axios in 2026?** No, not for basic scrapes. Node 18+ has a global `fetch`, and for single-file scripts it's less setup. Axios earns its place when you want interceptors, automatic retries, shared configurations or automatic JSON parsing across many requests.

**How do I know if a site needs JS rendering before writing code?** Fetch the page once and print the HTML. If your target data is in the raw string, Cheerio handles it. If not, check the Network tab for a JSON API you can call directly; only fall back to rendering (`render=true` or Playwright) when there's no clean endpoint.

**Is the free plan enough for a real project?** For learning and prototyping, yes — 1,000 credits with 5 concurrent connections covers a lot of static-page scraping. For anything sustained, Hobby at $49/month is the realistic entry point, and turning on auto-upgrade prevents silent pauses mid-job.

**What happens if I exceed my concurrent thread limit?** You get a 429 response and the request is rejected — no charge, no queuing. If you see 429s in your logs, either add a delay between requests or move up a tier.

**Does ScraperAPI handle the CAPTCHAs, or do I?** CAPTCHA and anti-bot detection handling is included on every plan, including the free one. Bypassing sites protected by Cloudflare, DataDome or PerimeterX adds 10 credits per successful bypass.

## Which setup should you actually pick

The cheapest correct answer, per scenario:

- **Static HTML targets, low volume, learning the craft** → built-in `fetch` + Cheerio, zero cost beyond your time. Don't buy anything yet.
- **Static targets, thousands of pages per month, or sites that fingerprint-block datacenter IPs** → Cheerio parsing kept as-is, fetch swapped for ScraperAPI. Hobby ($49/month, 100,000 credits) covers most personal projects; Startup ($149/month, 1M credits) is where production workloads usually land.
- **JS-rendered targets** → try the hidden-API route first; if there's no endpoint, `render=true` via ScraperAPI is simpler to operate than self-hosting Playwright, at 10 extra credits per page.
- **Heavily fortified sites at serious volume** → ScraperAPI's independent benchmark numbers (62–64% on hard targets) say look at the premium-priced competitors first, and verify against your own targets on a trial before paying anyone.

One last check before you subscribe: run your three most important target URLs through the domain cost estimator with the exact parameters you plan to use, and divide your plan's credits by that per-request cost. That single division tells you whether a tier can carry your workload — and it takes about two minutes with the 👉 [free 5,000-credit trial](https://www.scraperapi.com/signup?fp_ref=coupons) before any money changes hands.
