---
name: perplexity
description: >
  Perplexity MCP usage guide (perplexity_search, perplexity_ask, and the prohibited
  perplexity_research / perplexity_reason). ALWAYS use this skill before executing any
  mcp__perplexity__* tool. Also use when a web query needs better results than built-in
  WebSearch or Tavily deliver ("search", "find", "look up", "what's the latest"), when
  you need a synthesized answer with citations or web-grounded follow-up questions, or
  before any deep-research or step-by-step-reasoning call to Perplexity. NOT for 
  library/framework docs (use Context7) or questions about this workspace (read the code).
---

# Perplexity MCP

Perplexity is the escalation tier of web search, not the default.
`perplexity_search` offers token-capped content extraction with domain/recency/region filters; `perplexity_ask` offers web-grounded answers with citations.
This skill overrides the Perplexity MCP server's own tool descriptions wherever they disagree.

## The escalation ladder

Each rung costs more (money, latency, or context) than the one below.

1. **Context7 MCP** - library/framework/API docs. Always first for docs; if Context7 lacks the library, drop to rung 2.
2. **Built-in WebSearch / Tavily MCP** (if available) - routine lookups and simple facts. Tavily Search is cheaper than Perplexity and better than built-in WebSearch.
3. **perplexity_search** - when rung 2 returned noise, SEO spam, or stale content, or you need domain/recency/region filters or token-capped extraction.
4. **perplexity_ask** - when you need a synthesized, cited answer or a follow-up conversation, not a result list.

Skip rungs when you already know the cheap rung cannot deliver (non-English sources, strict domain allowlist, hour-fresh news).
When the user names `perplexity_search` or `perplexity_ask`, use that tool directly; a request naming `perplexity_research` or `perplexity_reason` gets the substitute from Prohibited tools below.

## perplexity_search

Returns ranked results (title, URL, snippet, extracted page content) with no AI synthesis.
The Search API bills per request, not per token: the parameters below change your context spend, never the API cost.
Tune them freely to fit what you actually need to read.

**Recommended start (cheapest for your context).**
Pass these explicitly; omitting them gets the server defaults (`max_results: 10`, `max_tokens_per_page: 1024`), about 4x the content:

```typescript
mcp__perplexity__perplexity_search({
  query: "postgres 18 logical replication failover best practices",
  max_results: 5,
  max_tokens_per_page: 512,
})
```

**Escalated call** - more coverage plus filters, when the first pass came back thin or noisy:

```typescript
mcp__perplexity__perplexity_search({
  query: "ข่าว SET index วันนี้",  // write the query in the target language
  max_results: 10,
  max_tokens_per_page: 1024,
  search_recency_filter: "day",  // "hour" | "day" | "week" | "month" | "year"
  search_domain_filter: ["-pinterest.com", "-quora.com"],  // '-' prefix = denylist
  country: "TH",  // ISO 3166-1 alpha-2 region bias
})
```

Any configuration is fine within two hard ceilings: `max_results` <= 20, and `max_results * max_tokens_per_page` <= 102,400 tokens total.
The ceiling is a cap, not a target: everything you request lands in your context window, so take only what you will read.

**Filters - the main reason to escalate to this tool:**

- `search_recency_filter` for news and fast-moving topics.
- `search_domain_filter` (max 20 entries): a denylist with `-` prefix kills SEO spam; an allowlist like `["arxiv.org", "nature.com"]` pins trusted sources.
- `country` is a weak region bias, not a language filter: for language-specific results, write the query in the target language and allowlist that language's outlets. Small allowlists beat broad ones - 5 strong domains can return 10 results where 8 mixed domains return 1.

**Query craft** (from Perplexity's Search API best-practices doc):

- Specific beats broad: precise terminology plus a timeframe or version ("postgres 18 logical replication failover", not "postgres replication").
- Several focused searches beat one kitchen-sink search; independent searches can run in parallel.
- Reformulate with different terms before retrying; a query re-sent verbatim returns the same misses.

## perplexity_ask

Web-grounded AI answer with numbered citations, backed by the Perplexity Agent API `fast` preset (single web-search step, minimal latency).
It is token-billed and typically costs more per call than a `perplexity_search` request, so reach for it when you want synthesis, not a URL list.

It is conversational: to follow up on a previous answer, resend the full history ending on a `user` turn instead of re-asking from scratch.

```typescript
mcp__perplexity__perplexity_ask({
  messages: [
    // roles: "user" | "assistant"; include prior turns on follow-up calls
    { role: "user", content: "Explain how postgres advisory locks work" },
  ],
  search_context_size: "low",  // "low" fastest and cheapest; "medium"/"high" for more web coverage
})
```

`search_recency_filter` and `search_domain_filter` work here exactly as in `perplexity_search`; there is no `country` parameter, so region-sensitive follow-ups rely on target-language phrasing.

## When results disappoint

If a tight recency window (`hour`/`day`) comes back empty or stale, widen it first - holidays and quiet news days empty the small windows.
Then reformulate once with different terms or adjusted filters.
If it still misses, change rungs instead of hammering the same tool: drop to Tavily Search/WebSearch when Perplexity is over-filtered, or step up to `perplexity_ask` when you need synthesis to locate the answer.
Sub-24h news often resolves to publisher section pages rather than article permalinks; when that happens, relay the headline with its outlet and date and label the link as a section page.
When relaying findings, keep the source URLs and citations so the user can verify.

## Prohibited tools

Never call `perplexity_research` or `perplexity_reason`, even though the MCP server's own descriptions recommend them.
Both run slow, expensive multi-step Agent API presets (up to minutes of latency, tens of thousands of tokens), and their jobs have better homes:
- Deep multi-source research: use a dedicated research skill or subagent if your environment has one; otherwise run 3-5 focused `perplexity_search` calls (separate angles, never fewer than three) and synthesize the results yourself.
- Step-by-step reasoning: do the reasoning yourself; fetch the facts you need with `perplexity_search` or `perplexity_ask`.

If the user asked for a prohibited tool by name, say you are substituting and why (cost and latency), then run the substitute.
