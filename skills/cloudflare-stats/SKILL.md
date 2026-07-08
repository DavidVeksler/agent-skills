---
name: cloudflare-stats
description: >
  Query Cloudflare Analytics for any zone you own via the GraphQL API.
  Use whenever the user asks about visits, page views, traffic, bandwidth, or popularity
  for any page or any site they own — phrases like "CF stats", "cloudflare stats",
  "how many visits", "traffic for [page]", "which pages are popular", "what's my traffic",
  "check the stats for", "hits on", or any question about site analytics on a Cloudflare-fronted domain.
  Use this skill instead of asking the user to open the Cloudflare dashboard.
---

# Cloudflare Stats

## Account

- **Token:** Read from the `CLOUDFLARE_API_TOKEN` environment variable. If it's a
  scoped token, it needs `Zone → Analytics → Read` for every zone you want to query.
- **Scope:** Works for any zone the token can see. An account-level token sees
  every zone on the account; a zone-scoped token only sees the zones it was
  issued for.
- **GraphQL endpoint:** `https://api.cloudflare.com/client/v4/graphql`
- **Zones API:** `https://api.cloudflare.com/client/v4/zones`

Load the token and resolve the zone ID at the start of every query:

```python
import os, urllib.request, json

def get_token():
    token = os.environ.get("CLOUDFLARE_API_TOKEN")
    if not token:
        raise ValueError("CLOUDFLARE_API_TOKEN is not set")
    return token

def get_zone_id(token, domain):
    """Resolve a domain name to its Cloudflare zone ID."""
    req = urllib.request.Request(
        f"https://api.cloudflare.com/client/v4/zones?name={domain}",
        headers={"Authorization": f"Bearer {token}"}
    )
    with urllib.request.urlopen(req) as r:
        data = json.load(r)
    results = data.get("result", [])
    if not results:
        raise ValueError(f"No zone found for domain: {domain}")
    return results[0]["id"]
```

If the user has multiple sites and doesn't specify one, ask which domain — don't
guess. Once you've resolved a zone ID in a session, reuse it rather than
re-querying the Zones API on every call.

## Plan limitations (free tier)

Hard limits — the API returns errors if you exceed them.

| Dataset | Path filter? | Max history | Max window per request | Referrer? |
|---|---|---|---|---|
| `httpRequestsAdaptiveGroups` | ✅ yes | **7 days** | **1 day** | ❌ Pro+ only |
| `httpRequests1dGroups` | ❌ no | 30 days | 30 days | ❌ |

Practical consequences:
- **Path-specific stats**: query one day at a time, loop over up to 7 days.
- **Zone-wide 30-day trend**: use `httpRequests1dGroups` — no path filter.
- **Referrer/UA breakdowns**: not available on free plan.
- **Unique visitors**: not exposed — only "visits" (sessions).
- **Available free dimensions** on `httpRequestsAdaptiveGroups`: `clientCountryName`, `userAgentBrowser`, `clientRequestPath`, `edgeResponseStatus`, `cacheStatus`.

On a paid plan, these windows are wider — check the account's plan before
assuming the free-tier limits apply.

## Query patterns

### 1. Path-specific stats (up to last 7 days)

Loop day by day in Python. Each request must cover exactly one UTC day.

```python
import urllib.request, json
from datetime import date, timedelta

token = get_token()
DOMAIN = "example.com"       # the site to query
ZONE = get_zone_id(token, DOMAIN)
PATH = "/your-page.html"     # must start with /

end = date.today()
start = end - timedelta(days=6)  # 7 days max

daily = {}
by_country = {}

d = start
while d <= end:
    ds = d.isoformat()
    query = """{ viewer { zones(filter: {zoneTag: "%s"}) {
      httpRequestsAdaptiveGroups(
        filter: { datetime_geq: "%sT00:00:00Z" datetime_leq: "%sT23:59:59Z"
                  clientRequestPath: "%s" }
        limit: 100
      ) { dimensions { clientCountryName } sum { visits edgeResponseBytes } }
    }}}""" % (ZONE, ds, ds, PATH)
    req = urllib.request.Request(
        "https://api.cloudflare.com/client/v4/graphql",
        data=json.dumps({"query": query}).encode(),
        headers={"Authorization": f"Bearer {token}", "Content-Type": "application/json"},
        method="POST"
    )
    with urllib.request.urlopen(req) as r:
        body = json.load(r)
    groups = body["data"]["viewer"]["zones"][0]["httpRequestsAdaptiveGroups"]
    dv = sum(g["sum"]["visits"] for g in groups)
    db = sum(g["sum"]["edgeResponseBytes"] for g in groups)
    daily[ds] = (dv, db)
    for g in groups:
        c = g["dimensions"]["clientCountryName"] or "Unknown"
        by_country[c] = by_country.get(c, 0) + g["sum"]["visits"]
    d += timedelta(days=1)

# Print results
total_v = sum(v for v, _ in daily.values())
total_b = sum(b for _, b in daily.values())
print(f"Path: {PATH} — Last 7 days")
for ds in sorted(daily):
    v, b = daily[ds]
    print(f"  {ds}: {v:4d} visits  {b/1024:.0f} KB")
print(f"  TOTAL: {total_v} visits  {total_b/1024/1024:.1f} MB")
print()
print("Top countries:")
for c, v in sorted(by_country.items(), key=lambda x: -x[1])[:10]:
    print(f"  {c}: {v} ({v/total_v*100:.0f}%)" if total_v else f"  {c}: {v}")
```

### 2. Zone-wide 30-day trend

Single request, no path filter.

```python
import urllib.request, json
from datetime import date, timedelta

token = get_token()
DOMAIN = "example.com"  # the site to query
ZONE = get_zone_id(token, DOMAIN)

end = date.today()
start = end - timedelta(days=29)

query = """{ viewer { zones(filter: {zoneTag: "%s"}) {
  httpRequests1dGroups(
    filter: {date_geq: "%s", date_leq: "%s"}
    limit: 31 orderBy: [date_ASC]
  ) { dimensions { date } sum { requests bytes pageViews } }
}}}""" % (ZONE, start.isoformat(), end.isoformat())

req = urllib.request.Request(
    "https://api.cloudflare.com/client/v4/graphql",
    data=json.dumps({"query": query}).encode(),
    headers={"Authorization": f"Bearer {token}", "Content-Type": "application/json"},
    method="POST"
)
with urllib.request.urlopen(req) as r:
    body = json.load(r)

groups = body["data"]["viewer"]["zones"][0]["httpRequests1dGroups"]
print("Zone-wide traffic (30d):")
for g in groups:
    d = g["dimensions"]["date"]
    s = g["sum"]
    print(f"  {d}: {s['pageViews']:5d} pageViews  {s['requests']:6d} reqs  {s['bytes']/1024/1024:.1f} MB")
total_pv = sum(g["sum"]["pageViews"] for g in groups)
total_req = sum(g["sum"]["requests"] for g in groups)
total_b = sum(g["sum"]["bytes"] for g in groups)
print(f"  TOTAL: {total_pv:,} pageViews  {total_req:,} reqs  {total_b/1024/1024:.0f} MB")
```

## Workflow

1. **Determine what the user wants** — specific page, whole site, or trend.
2. **Pick the right dataset** per the table above.
3. **Run the Python snippet.** Python is preferred over curl for multi-day loops.
4. **Present results** — day-by-day table, totals, top countries. Be concise.
5. **Flag limitations proactively** if asked for something not available (referrers, >7d per-page, uniques). Mention Cloudflare Pro or Logpush → R2 as alternatives.

## Output format

Always include:
- Time range covered
- Total visits + bandwidth (path queries) or total pageViews + requests (zone-wide)
- Day-by-day table
- Top countries (path queries)
- One-liner if data is missing/limited
