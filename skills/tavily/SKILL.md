---
name: tavily
description: >
  Tavily MCP usage guide (tavily_search, tavily_extract, tavily_map, tavily_crawl, and
  the prohibited tavily_research). ALWAYS use this skill before executing any
  mcp__tavily-remote-mcp__* tool. Also use for web queries ("search", "find", "look up",
  "what's the latest"), reading the content of specific URLs, crawling or bulk-reading a
  site's pages, mapping a site's structure, or before any deep-research call to Tavily.
  NOT for library/framework docs (use Context7 first) or questions about this workspace
  (read the code).
---

# Tavily MCP

Tavily is the workhorse web tool: better results than built-in WebSearch, cheap to run (1,000 free credits/month, then $0.008/credit).
Four tools with one job each: **search finds pages, extract reads them, map surveys a site, crawl harvests one**.
This skill overrides the Tavily MCP server's own tool descriptions wherever they disagree.

## Picking the tool

| You have / you need | Tool |
|---|---|
| A question, no URLs | `tavily_search` |
| URLs, need their content | `tavily_extract` |
| A site, need its URL list or structure | `tavily_map` |
| A site, need many pages' content | `tavily_crawl` (map first) |

- Library/framework/API docs: Context7 first; if it lacks the library, fall through to `tavily_search`.
- Throwaway lookups where result quality barely matters (a definition, a quick sanity check): built-in WebSearch costs nothing.
- The primary pattern is **search then extract**: search to discover sources, then extract the top hits by relevance score (up to 5 URLs bill as one credit).
- When the user names a specific Tavily tool, skip the picking step but keep that tool's parameter guardrails; a request naming `tavily_research` gets the substitute from Prohibited tool below.

## Credits

All numbers come from Tavily's API-credits doc; billing blocks round up (12 extracted URLs = 3 credits).

| Call | Credits |
|---|---|
| `tavily_search` (basic / fast / ultra-fast) | 1 per request |
| `tavily_search` (advanced) | 2 per request |
| `tavily_extract` | 1 per 5 successful URLs (advanced: 2 per 5) |
| `tavily_map` | 1 per 10 pages (with `instructions`: 2 per 10) |
| `tavily_crawl` | map cost + extract cost, about 3 per 10 pages at basic depth |

Search bills per request: `search_depth: "advanced"` doubles that request's price, and every other search parameter only changes your context spend.
Extract, map, and crawl bill by volume: URL count, `limit`, `max_depth`, and `max_breadth` are the real cost levers, multiplied by depth upgrades and `instructions`.
Failed extractions and failed maps are not billed.

## tavily_search

Returns ranked results with title, URL, relevance score, and a content snippet.
The schema defaults are already the cheap path (`search_depth: "basic"`, `max_results: 5`), and every filter is free at any depth:

```typescript
mcp__tavily-remote-mcp__tavily_search({
  query: "Thai stock market news",
  time_range: "day",  // "day" | "week" | "month" | "year"
  country: "Thailand",  // full country name, NOT an ISO code like "TH"
})
```

Escalate only when the basic pass came back thin, noisy, or stale: `search_depth: "advanced"` (2 credits, better relevance and richer snippets), plus `max_results: 10` if you need more coverage - only the depth change costs extra.

**Parameter notes:**

- `search_depth`: `"basic"` first in almost every case; jump straight to `"advanced"` only for a deliberate multi-search sweep like the research substitute below; `"fast"`/`"ultra-fast"` only when latency beats relevance.
- `include_domains` pins trusted sources; `exclude_domains` (e.g. `["pinterest.com", "quora.com"]`) kills SEO spam; `start_date`/`end_date` (YYYY-MM-DD) for precise windows; `exact_match` to require quoted phrases.
- News queries drown in social-media reposts even with `time_range: "day"`: pin `include_domains` to real outlets, or exclude the social platforms (`instagram.com`, `tiktok.com`, `facebook.com`). For sources in a specific language, pin that language's outlets and write the query in it - `country` is only a bias and no language parameter exists.
- `time_range` is approximate: before presenting anything as within 24 hours, verify freshness from the results' own dates (URL paths, published dates).
- `topic` is locked to `"general"` on this server; use `time_range` or dates for news recency.
- `include_raw_content` is free but dumps full page text for every result into your context; extracting the few URLs you actually chose costs one credit and loads far less.

**Query craft** (from Tavily's agent guide):

- Keep queries focused and under 400 characters; precise terminology plus a timeframe or version beats a broad phrase.
- Break complex topics into several focused sub-queries; independent searches can run in parallel.
- Reformulate with different terms before retrying; a query re-sent verbatim returns the same misses.

## tavily_extract

Turns URLs into clean markdown (`format` defaults to `"markdown"`), up to 20 URLs per call - split larger sets into batches of 20.

```typescript
mcp__tavily-remote-mcp__tavily_extract({
  urls: ["https://example.com/docs/replication"],
  query: "failover steps",  // optional: reranks content chunks so mostly-relevant parts return
})
```

- Start with the default `extract_depth: "basic"`; switch to `"advanced"` (double cost) when content comes back incomplete or the site is JS-heavy, protected, or table-dense.
- Batch URLs into one call: billing rounds up per 5 successful URLs, so 5 URLs cost the same as 1.
- Set `query` whenever you need part of a long page rather than all of it.

## tavily_map and tavily_crawl

`tavily_map` returns a site's URL list; `tavily_crawl` is map plus extraction in one operation.
Both bill by page volume, and crawl is the most expensive tool here, so survey before you harvest: map first, then either extract the handful of URLs you actually need or crawl with tight bounds.

```typescript
mcp__tavily-remote-mcp__tavily_map({
  url: "https://docs.example.com",
  limit: 30,  // map bills per 10 pages too; the default 50 runs up to 5 credits
})

mcp__tavily-remote-mcp__tavily_crawl({
  url: "https://docs.example.com",
  max_depth: 1,  // raise only when the map showed deeper structure you need
  limit: 20,  // total pages; bound it to what you will actually read
  select_paths: ["/api/.*"],  // regex filter - free, prefer it over instructions
})
```

- `max_breadth` (default 20) caps links followed per page; `select_domains` (regex) bounds a site spread across subdomains.
- `instructions` (natural-language page filter) doubles the mapping cost; reserve it for filters a path regex cannot express.
- Everything a crawl returns lands in your context window as well as on the bill.

## When results disappoint

An empty results array is a silent miss, not an error: reformulate with different terms or widen `time_range` first (holidays and quiet days empty the `"day"` window).
Otherwise escalate depth or reformulate once (each tool's notes above); if Tavily is unavailable or still misses, fall back to built-in WebSearch - free and always present - rather than hammering the same call.
When relaying findings, keep the source URLs so the user can verify.

## Prohibited tool

Never call `tavily_research`, even though the MCP server's own description recommends it.
A single call bills 4-250 credits (dynamic pricing) and can run for minutes, and its job has a better home:

- Deep multi-source research: use a dedicated research skill or subagent if your environment has one; otherwise run the search-then-extract pipeline (3-5 focused searches, extract the best sources) and synthesize the results yourself.
- If the user asked for `tavily_research` by name, say you are substituting and why (cost and latency), then run the substitute.
