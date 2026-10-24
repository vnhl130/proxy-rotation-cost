# proxy rotation python: rotating gateways, requests code that actually works, and the real per-request cost math

Most people searching this are stuck at the same fork: build a rotating pool yourself in Python, or push every request through a service that rotates for you. Both work. The right answer depends on your targets, your volume, and how much failure-handling code you want to own. This piece walks through the DIY path with `requests`, shows where it falls apart at scale, and then covers what a managed rotating proxy endpoint (ScraperAPI) actually costs once you account for how its credits are billed.

**Quick picks**

- Just need a few thousand requests to easy targets? The DIY pool below is fine, and a free tier covers testing: ScraperAPI gives 1,000 free credits on signup and a 7-day trial with 5,000 credits, no credit card required.
- Scraping Amazon, Google, or sites behind Cloudflare/Datadome at any real volume? A rotating gateway will save you days of maintenance, and paid plans start at $49/month (or $44.10/month billed annually).
- 👉 [Check ScraperAPI's free trial and current plans](https://www.scraperapi.com/?fp_ref=coupons)

## Rotate proxies in Python with requests: the 20-line version

The core idea is simple: keep a list of proxy URLs and pick a different one per request. `requests` takes a per-scheme dictionary, and the same URL usually works for both keys, because the client opens a CONNECT tunnel for HTTPS through the HTTP proxy.

python
import itertools, requests

PROXIES = [
    "http://user:pass@proxy1.example:8000",
    "http://user:pass@proxy2.example:8000",
    "http://user:pass@proxy3.example:8000",
]
proxy_cycle = itertools.cycle(PROXIES)

for url in urls:
    proxy = next(proxy_cycle)
    resp = requests.get(url, proxies={"http": proxy, "https": proxy}, timeout=10)


`random.choice` instead of `itertools.cycle` is the other common picker. Round-robin spreads load evenly across a small pool; random selection avoids a predictable request order, which matters because an entire pool can get fingerprinted as a group. In practice, weighting picks by each proxy's recent success rate beats both.

This version works. It also stops working the moment your targets get serious, for reasons the code above doesn't show.

## The three bugs that kill DIY rotation at scale

These are the failure modes that barely get explained in most tutorials, and they're the reason homegrown rotators quietly leak data:

1. **Session pinning.** Setting `session.proxies` on a `requests.Session` routes *every* request through that session to the same exit. Per-request rotation means passing `proxies=` on each call, or keeping one session per proxy so the connection pool and cookies stay aligned with the IP. Mixing a single cookie jar across many exits is a strong bot signal — worse than not rotating at all.
2. **Environment variable overrides.** `requests` also honors the `HTTP_PROXY` and `HTTPS_PROXY` environment variables, so a stray shell export can silently override the dict you pass in during debugging.
3. **Naive failure handling.** Two rules separate working rotators from broken ones: never delete an IP on a single failure, and never treat all failures the same. A connection reset is a proxy problem. A 403 with a challenge page is a target problem (the IP may be fine tomorrow). A 407 is your own credentials. Score only the first two against the proxy, give every exit a cooldown (300 seconds is a reasonable default in published examples) instead of deleting it, and persist that state so a restart doesn't reset everything you've learned.

Add a per-domain rate limiter too: 20 concurrent requests from 20 different IPs to one small site is still 20 concurrent requests. Concurrency without cooldowns just burns the pool faster.

If you're on Scrapy, don't write any of this by hand — the `scrapy-rotating-proxies` middleware handles rotation plus ban detection, tracks proxy state, and rechecks dead proxies automatically.

## When DIY stops making sense

There's a useful way to frame the decision. Proxy rotation setups fall into three models:

|  | Client-side list | Rotating gateway | Pay-per-success scrape API |
| --- | --- | --- | --- |
| What you write | Pool loader, picker, health checker, cooldown logic, retry loop | One proxy URL, optional session ID | One request per URL |
| Where rotation state lives | Your process (lost on restart unless persisted) | Provider side | Provider side |
| Ban handling | You classify 403/429/captcha and retire IPs | Fresh IP on next request; you still retry | Retries handled; failed calls not billed |
| Cost shape | Per GB, including blocked responses | Per GB | Per successful page |
| Fits | Owned proxy lists, odd protocols, full control | High-volume scraping across many targets | Small-to-medium volume, no infra |

Notice the cost rows. Self-managed pools bill per GB *including blocked responses* — you pay for the traffic that got you a captcha. Pay-per-success APIs bill per successful page, which is the only cost figure that lets you fairly compare a DIY pool against a managed service.

Most production setups end up using two of these at once: a gateway or API for hard targets, and a plain list for internal or tolerant endpoints.

## How ScraperAPI fits into Python rotation

ScraperAPI is the gateway/API column of that table. It handles IP rotation, CAPTCHA handling, and retries server-side, and claims a pool of 40M+ proxies across 50+ countries with ML-based selection of IP and header combinations per request. Two ways to use it from Python:

**1. The API endpoint.** Send a GET request to the API with your key, target URL, and optional `country` parameter for geotargeting:

python
import requests

resp = requests.get(
    "https://api.scraperapi.com",
    params={"api_key": API_KEY, "url": target, "country": "us"},
    timeout=60,
)
print(resp.status_code, resp.text)


**2. The proxy port.** If you already have code built around a proxy dict, this is the drop-in path — ScraperAPI exposes a proxy front-end that behaves like any other proxy, with username `scraperapi` and your API key as the password:

python
proxies = {
    "http": f"http://scraperapi:{API_KEY}@proxy-server.scraperapi.com:8001",
    "https": f"http://scraperapi:{API_KEY}@proxy-server.scraperapi.com:8001",
}
r = requests.get("http://httpbin.org/ip", proxies=proxies, verify=False)


Two details from the official docs worth knowing before you ship this:

- Your code must be configured to not verify SSL certificates in proxy mode, so requests route correctly through the API. If you need SSL verification, you can download ScraperAPI's Proxy CA certificate and trust it manually — the docs walk through installing it on Windows (via the Microsoft Management Console), macOS (Keychain Access), and Linux.
- Extra features turn on through the proxy username, separated by periods: enable JS rendering by setting the username to `scraperapi.render=true`, and combine multiple parameters the same way.

The proxy port matters for a specific workflow: if you've already built session logic, cookie handling, or Selenium integration around a proxy dict, you keep all of it and only swap the credentials. Selenium Wire, for instance, just points at ScraperAPI's endpoint and inherits the rotation and retry behavior.

One more thing to know about concurrency before you pick a plan: if you exceed your plan's thread limit, you get a 429 asking you to bring concurrency back within limits and retry. Per the pricing page: "There's no extra charge for going over, the request is simply turned away rather than queued." Overages cost nothing; your code just needs a retry.

## ScraperAPI plans: the full price sheet

All plans include JS rendering, premium residential and mobile IPs, rotating proxy pools, custom headers and sessions, CAPTCHA handling, automatic retries, unlimited bandwidth, and a 99.9% uptime guarantee. Annual billing takes 10% off every plan.

| Plan | Monthly | Annual (per month) | Credits | Concurrent threads |
| --- | --- | --- | --- | --- |
| Hobby | $49 | $44.10 | 100,000 | 20 |
| Startup | $149 | $134.10 | 1,000,000 | 50 |
| Business | $299 | $269.10 | 3,000,000 | 100 |
| Scaling (most popular) | $475 | $427.50 | 5,000,000 | 200 |
| Professional | $975 | $877.50 | 10,500,000 | 300 |
| Advanced | $1,975 | $1,777.50 | 21,500,000 | 500 |
| Enterprise | Custom quote | Custom quote | 22,000,000+ | 500+ |

Geotargeting differs by tier: Hobby and Startup cover US and EU only, with country-level targeting starting at Business. Analytics history is 30 days on the lower tiers and unlimited from Business up.

Every plan's purchase button lives on the pricing page, so all rows link there:

- Hobby: 👉 [Start with Hobby — $49/month](https://www.scraperapi.com/pricing?fp_ref=coupons)
- Startup: 👉 [See Startup plan details](https://www.scraperapi.com/pricing?fp_ref=coupons)
- Business: 👉 [View Business plan pricing](https://www.scraperapi.com/pricing?fp_ref=coupons)
- Scaling: 👉 [Check the Scaling plan](https://www.scraperapi.com/pricing?fp_ref=coupons)
- Professional: 👉 [View Professional plan pricing](https://www.scraperapi.com/pricing?fp_ref=coupons)
- Advanced: 👉 [Check the Advanced plan](https://www.scraperapi.com/pricing?fp_ref=coupons)
- Enterprise: 👉 [Talk to sales about Enterprise](https://www.scraperapi.com/pricing?fp_ref=coupons)

A second table, because these limits decide which tier you actually land on:

|  | Hobby / Startup | Business and up |
| --- | --- | --- |
| Geotargeting | US & EU only | Country-level |
| Analytics history | 30 days | Unlimited |
| Auto-upgrade when credits run out | Yes (opt-in) | Yes, plus pay-as-you-go on Scaling+ |

## The real cost math most reviews skip

Here's the part that pricing-page screenshots never show: a "credit" is one standard page, but your targets don't bill that way. Per the official pricing page:

> A standard page costs 1 credit. Amazon costs 5, Google and Bing (including their subdomains) cost 25, and LinkedIn costs 30. Sites behind bot protection like Cloudflare, Datadome, or PerimeterX add 10 credits per request when we bypass them.

Run the numbers on the Hobby plan and this changes everything. 100,000 credits is 100,000 standard pages — but it's 20,000 Amazon pages, 4,000 Google searches, or roughly 3,300 LinkedIn pages. A bot-protection bypass adds 10 credits *on top of* the page cost, so a Google page behind Cloudflare-style protection could run 35 credits.

Two tools keep this under control, both built into the dashboard:

- **Domain Cost Estimator** — check the exact credit cost for any URL before you build your scraper around it.
- **`max_cost` per scrape** — set a hard ceiling on any single request so one accidental LinkedIn crawl doesn't drain a week of credits.

There's a decision rule hiding in these numbers: if your project is mostly standard pages (blogs, e-commerce listings, documentation), Hobby's 100,000 credits go a long way. If your targets are search engines, LinkedIn, or heavily protected sites, your effective cost per page is 5–35x the sticker rate, and you should budget on those numbers — Hobby might only be a few thousand useful requests, which pushes you toward Startup or Business. You can estimate this before committing any money.

When credits run out mid-project, scraping pauses by default until renewal. Hobby through Professional can opt into auto-upgrade, which bumps you to the next plan automatically. Scaling and above can also use pay-as-you-go at a fixed per-credit rate, capped at a monthly limit you set, so a spike doesn't turn into a surprise invoice.

## Testing before you pay

You can validate the whole setup without a credit card. Signup gets you a free plan with 1,000 API credits (maximum 5 concurrent connections), and there's a 7-day free trial with 5,000 credits on top. If you need more credits just for testing, support can grant additional ones.

That's enough to run the domain cost estimator against your actual targets and confirm what your crawl would really cost per page — which is the single most useful thing to do before choosing between Hobby and anything bigger. 👉 [Create a free ScraperAPI account and get 5,000 trial credits](https://dashboard.scraperapi.com/signup?fp_ref=coupons)

## FAQ

**Do I need to rotate proxies myself if I use a rotating gateway?**
No. The gateway rotates IPs server-side on its end; your code just sends requests to one stable endpoint. You still own retry logic for transient failures, but you no longer maintain a pool, health checks, or cooldown state.

**How many proxies do I need for Python scraping?**
There's no fixed number — what matters is requests per second against your hardest target and how quickly that target bans. A managed pool with 40M+ IPs across 50+ countries sidesteps the question entirely, which is one reason teams move off self-managed lists once volume grows.

**What happens if my request fails through ScraperAPI?**
Retries and CAPTCHA handling are handled by the service on the API path. On the pricing side, you're billed per page requested according to the domain cost table, so it's worth checking any unfamiliar domain in the estimator before you point a large crawl at it.

**Is annual billing worth it?**
If you'll use the service longer than a few months, yes — annual billing takes 10% off every plan tier, dropping Hobby from $49 to $44.10 per month and Scaling from $475 to $427.50. Month-to-month keeps flexibility if you're still validating the project.

## Picking a plan based on what you're actually scraping

- **Learning or building a prototype:** the free plan (1,000 credits, 5 concurrent connections) and the 7-day/5,000-credit trial will carry you through. Don't pay yet.
- **Standard pages — blogs, docs, product listings without heavy protection:** Hobby at $49/month (100,000 credits, 20 threads) is the right entry point, and annual pricing takes it to $44.10.
- **Search engines, LinkedIn, or Cloudflare/Datadome-protected targets:** price your project in effective credits first (25–35 per hard page), then pick the tier — most serious projects on these targets land on Startup ($149) or Business ($299) with country-level geotargeting.
- **Multi-step flows that need the same IP across a session:** custom sessions are supported on all plans, and the proxy port keeps your existing session logic intact.
- **Enterprise-scale (22M+ credits per month):** that's a custom quote conversation, not a checkout button.

One last check before you commit: run your top three target URLs through the domain cost estimator, multiply by your monthly volume, and compare that against the credit allowance of the plan you're eyeing. If the math lands inside Hobby, start there — you can upgrade from the billing page anytime. 👉 [See ScraperAPI's full plan lineup and start free](https://www.scraperapi.com/pricing?fp_ref=coupons)
