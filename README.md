# Cloudflare Bypass API: What ScraperAPI Charges, Which Plans Fit, and When a Bypass API Won't Help

ScraperAPI handles Cloudflare-protected pages through the same HTTPS endpoint you already use for normal pages, and it bills the bypass at **10 credits per request** on top of the base cost. A Hobby plan at **$49/month for 100,000 credits** therefore covers about 10,000 Cloudflare-bypassed pages, not 100,000. The free tier gives you 1,000 credits and 5 concurrent connections to test whether your target site clears at all.

Below: every plan with its real price, how credits actually get eaten by protected pages, the difference between an API that returns HTML and one that hands you a `cf_clearance` cookie, and the protection types ScraperAPI's bypass does not cover.

## Quick pick

Most developers scraping a handful of Cloudflare-fronted sites should start on **Hobby at $49/month** (100,000 credits, 5 concurrent connections) and use the free 1,000-credit tier first to confirm the target clears. Move to Startup only when you need more concurrency or volume.

👉 [Check the current Hobby plan limits and start with the free 1,000 credits](https://www.scraperapi.com/pricing?fp_ref=coupons)

## All ScraperAPI plans

Every plan below is billed monthly in USD and includes the same core feature set: JS rendering, premium proxies, geotargeting, sessions, and bot-protection bypass, all paid for in credits. Annual billing takes **10% off** across Hobby through Advanced; Enterprise is custom-quoted.

| Plan | Credits / month | Concurrent connections | Price (monthly) | Best for | Get started |
| --- | --- | --- | --- | --- | --- |
| Free | 1,000 | 5 | $0 | Confirming a target site clears before paying | [Start free](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Hobby | 100,000 | 10 | $49 | Solo developers, a few protected sites, low volume | [Get Hobby](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Startup | 500,000 | 25 | $149 | Small teams running scheduled jobs | [Get Startup](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Business | 3,000,000 | 50 | $299 | Production pipelines with steady daily volume | [Get Business](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Scaling | 10,000,000 | 100 | $475 | High-volume scraping with pay-as-you-go overflow | [Get Scaling](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Professional | 20,000,000 | 200 | $975 | Teams that need both auto-upgrade and PAYG | [Get Professional](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Advanced | 40,000,000 | 300 | $1,975 | Large-scale operations, no auto-upgrade | [Get Advanced](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Enterprise | Custom | Custom | Custom | Custom SLAs, discounted custom deals | [Talk to sales](https://www.scraperapi.com/pricing?fp_ref=coupons) |

A few things the table can't show:

- **Credits do not roll over.** Your balance resets at each renewal. Unused credits are gone.
- **Pay-as-you-go is not available on Hobby, Startup, or Business.** If you run out mid-month on those plans, requests stop until renewal unless you opt into auto-upgrade, which bumps you to the next plan.
- **Auto-upgrade is opt-in** on Hobby, Startup, Business, Scaling, and Professional, and unavailable on Advanced and Enterprise.
- **Exceeding your concurrency limit returns a 429**, not an extra charge. The request is rejected rather than queued.
- **Annual plans can renew early** to get a fresh credit batch.

## How Cloudflare pages consume credits

This is the part that decides whether your monthly bill matches your estimate, and it's the part most write-ups skip.

ScraperAPI prices protected domains separately from normal ones. The baseline is **1 credit for a standard page**. A page behind Cloudflare, Cloudflare Turnstile, DataDome, PerimeterX/HUMAN, or Akamai adds **10 credits per request** when the bypass mechanism activates. So one successfully unblocked Cloudflare page costs **11 credits**, not 1.

Stack parameters on top and it compounds:

| Request type | Credits per request |
| --- | --- |
| Standard page | 1 |
| Cloudflare / Turnstile / DataDome / PerimeterX bypass | +10 |
| JS rendering (`render=true`) | +10 |
| Premium proxy (`premium=true`) | +10 |
| Screenshot (`screenshot=true`) | +10 |
| Premium proxy + JS rendering | 25 |
| Ultra premium (`ultra_premium=true`) | 30 (paid plans only) |
| Ultra premium + JS rendering | 75 (paid plans only) |

A Cloudflare-protected page that also needs JavaScript rendering and a premium proxy runs **31 credits**: 1 base + 10 bypass + 10 render + 10 premium. On Hobby's 100,000 credits that's roughly 3,200 pages a month, not 10,000. If your target is a JS-heavy single-page app behind Cloudflare, plan your tier around that number.

Some parameters cost nothing extra: `country_code`, `session_number`, `device_type`, `output_format`, `wait_for_selector`, `keep_headers`, and `autoparse`. Geotargeting in particular is free, which matters if you need to see a page as a user in a specific country to get the right pricing or content.

Every response carries an `sa-credit-cost` header telling you what that request actually cost. Before you commit to a volume estimate, call the cost endpoint with your real URL and parameters, or use the Domain Cost Estimator in the dashboard. You can also set `max_cost` per scrape so a single unexpectedly expensive URL can't blow through your budget.

👉 [See the full credit-cost table and start with 1,000 free credits](https://www.scraperapi.com/pricing?fp_ref=coupons)

## What "bypass" means here versus building your own

Two different products get sold under the phrase "Cloudflare bypass API," and confusing them is how people end up with a bill or a broken pipeline.

**A hosted scraping API returns the page.** You send a URL, you get back HTML, a status code, and a cost header. ScraperAPI works this way. So does the Cloudflare Bypass and Website Unblocker actor on Apify, which returns one record per URL with fields like `success`, `statusCode`, `protectionDetected`, `pageTitle`, `contentLength`, `html`, and `attempts`. In a published sample record for a Cloudflare challenge page, that actor reported `attempts: 6` before succeeding — six retrieval attempts, one delivered page. That attempt count is worth knowing about because it's invisible in most marketing copy: solving a challenge is a retry loop, not a single round trip.

**A self-hosted bypass server returns a cookie.** Open-source projects like Unflare and chrome-bypass-server run a headless browser, solve the challenge, and hand back the `cf_clearance` cookie plus the matching user agent and headers, so your own scraper can make subsequent requests without triggering the challenge again.

The catch with the second approach is IP binding. `cf_clearance` is tied to the IP and user agent that solved the challenge. chrome-bypass-server's documentation states the intended usage plainly: applications **behind the same IP** ask the server for a bypass. If your scraper runs on a different egress IP than the bypass server, the cookie you get back is worthless. Unflare addresses this with a `proxy` parameter (`host`, `port`, `username`, `password`) so you can pin the solving browser to a specific exit.

That's where the "free and open source" math breaks down. A self-hosted bypass server needs:

- A host running Chrome, plus `Xvfb` on Linux if you're not using Docker (Unflare's Docker image bundles the dependencies and listens on port 5002)
- A residential or sticky proxy so the solving IP matches your request IP, and stays stable enough for the cookie to remain valid
- Operational attention: Unflare writes screenshots to `/screenshots` inside the container on errors or blocks, which is helpful for debugging and useless if nobody checks them

A hosted API folds all of that into a per-request price. At 11 credits per Cloudflare page on Hobby, that's roughly **$0.0054 per page** at the plan's effective rate — cheap against a proxy bill, expensive if a bug in your code is firing requests you didn't need.

The reverse trade-off is real too. Self-hosting gives you the cookie, which means you can reuse one solved session across many requests to the same domain. Hosted APIs charge per request, so a thousand calls to one domain cost a thousand bypass fees. If your workload is deep crawling of a small number of Cloudflare-protected domains, a cached cookie is dramatically cheaper. If your workload is wide — many domains, each visited a few times — the hosted API wins on both engineering time and predictability.

## Where the bypass stops working

ScraperAPI lists Cloudflare, Cloudflare Turnstile, DataDome, PerimeterX/HUMAN, and Akamai as covered protections, each at 10 credits. That's a broader list than most self-hosted projects advertise.

What no bypass API in this category handles well is interactive human verification. The Apify actor's documentation is explicit that it does not clear hard CAPTCHAs, Imperva/Incapsula, or PerimeterX — and it does not charge for those failures. Treat that as a general characteristic of hosted bypass tooling rather than something specific to one vendor: an interstitial challenge that resolves itself in a browser is automatable; a puzzle that requires a human is a different problem, and you should expect a `success: false` rather than a silent failure or an infinite charge.

Before you buy a plan, hit your actual target URL once and check what comes back. If the response is a challenge page or an error rather than content, more credits won't fix it — you need a different tool or a different data source entirely. Testing on the free tier costs nothing and takes a few minutes.

👉 [Test your target URL on the free plan before paying for credits](https://www.scraperapi.com/pricing?fp_ref=coupons)

## Which plan to buy

Work backwards from credits per page, not pages per month. The number that matters is your per-request credit cost, and for Cloudflare work it's rarely 1.

- **A few thousand protected pages a month, no JS rendering** → Hobby, $49. At 11 credits per page you have room for about 9,000.
- **The same volume but JS-heavy pages behind Cloudflare** → still Hobby, but at 21 credits per page your ceiling drops to roughly 4,700. If that's tight, the 10% annual discount is the cheapest way to buy headroom.
- **Scheduled jobs that must not stop mid-month** → Scaling at $475 or Professional at $975, the tiers where pay-as-you-go is available. On Hobby, Startup, or Business your only overflow option is auto-upgrade to the next plan.
- **Concurrency above 50** → Business and up. A 429 on a rejected request doesn't cost credits, but it does cost you the run.
- **Custom pricing or a discounted deal** → Enterprise. Note that custom-deal accounts carry a higher credit cost per request on ultra-premium domains, to hold a minimum price of $3 per thousand requests without rendering and $7 with rendering. Worth asking about before you sign if ultra-premium domains are in scope.

Two things to confirm before you enter a card: whether your specific target domains clear (the free tier answers this), and what your effective per-page cost is after parameters, since `render=true` doubles the price of an already-expensive bypassed request. Set `max_cost` on your first production jobs so a misconfigured parameter can't quietly multiply your bill.

👉 [Compare every plan side by side and pick your credit tier](https://www.scraperapi.com/pricing?fp_ref=coupons)
