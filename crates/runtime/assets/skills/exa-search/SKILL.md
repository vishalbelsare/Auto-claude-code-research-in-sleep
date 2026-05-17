---
name: exa-search
description: AI-powered web search via Exa with content extraction. Use when user says "exa search", "web search with content", "find similar pages", or needs broad web results beyond academic databases (arXiv, Semantic Scholar).
argument-hint: [search-query-or-url]
allowed-tools: Bash(*), Read, Write
---

# Exa AI-Powered Web Search

Search query: $ARGUMENTS

## Role & Positioning

Exa is the **broad web search** source with built-in content extraction:

| Skill | Best for |
|------|----------|
| `/arxiv` | Direct preprint search and PDF download |
| `/semantic-scholar` | Published venue papers (IEEE, ACM, Springer), citation counts |
| `/deepxiv` | Layered reading: search, brief, section map, section reads |
| `/exa-search` | Broad web search: blogs, docs, news, companies, research papers — with content extraction |

Use Exa when you need results beyond academic databases, or when you want content (highlights, full text, summaries) extracted alongside search results.

## Constants

- **FETCH_SCRIPT** — `tools/exa_search.py` relative to the current project.
- **MAX_RESULTS = 10** — Default number of results to return.

> Overrides (append to arguments):
> - `/exa-search "RAG pipelines" — max: 5` — top 5 results
> - `/exa-search "diffusion models" — category: research paper` — research papers only
> - `/exa-search "startup funding" — category: news, start date: 2025-01-01` — recent news
> - `/exa-search "transformer" — content: text, max chars: 8000` — full text mode
> - `/exa-search "transformer" — content: summary` — LLM-generated summaries
> - `/exa-search "transformer" — domains: arxiv.org,huggingface.co` — domain filter
> - `/exa-search "https://arxiv.org/abs/2301.07041" — similar` — find similar pages

## Setup

Exa requires the `exa-py` SDK and an API key:

```bash
pip install exa-py
```

Set your API key:
```bash
export EXA_API_KEY=your-key-here
```

Get a key from [exa.ai](https://exa.ai).

## Workflow

### Step 1: Parse Arguments

Parse `$ARGUMENTS` for:
- **query**: The search query (required) or a URL (for `find-similar` mode)
- **similar**: If present, use `find-similar` mode instead of search
- **max**: Override MAX_RESULTS
- **category**: `research paper`, `news`, `company`, `personal site`, `financial report`, `people`
- **content**: `highlights` (default), `text`, `summary`, `none`
- **max chars**: Max characters for content extraction
- **type**: Search type — `auto` (default), `neural`, `fast`, `instant`
- **domains**: Comma-separated include domains
- **exclude domains**: Comma-separated exclude domains
- **include text**: Phrase that must appear in results
- **exclude text**: Phrase to exclude from results
- **start date**: ISO 8601 date — only results after this
- **end date**: ISO 8601 date — only results before this
- **location**: Two-letter ISO country code

### Step 2: Locate Script (Policy A — gate)

Resolve via the standard fallback chain (`shared-references/integration-contract.md` §1). Missing helper aborts this skill because there is no graceful alternative search backend within /exa-search.

```bash
SCRIPT=""
# Layer 2: user-customised
[ -n "${HOME:-}" ] && [ -f "$HOME/.config/aris/tools/exa_search.py" ] && SCRIPT="$HOME/.config/aris/tools/exa_search.py"
# Layer 3: aris-code bundled cache (v0.4.8+)
[ -z "$SCRIPT" ] && [ -n "${ARIS_CACHE_DIR:-}" ] && [ -f "$ARIS_CACHE_DIR/tools/exa_search.py" ] && SCRIPT="$ARIS_CACHE_DIR/tools/exa_search.py"
# Layer 4: project workspace
[ -z "$SCRIPT" ] && [ -f "tools/exa_search.py" ] && SCRIPT="tools/exa_search.py"
# Legacy: ~/.claude/skills/
[ -z "$SCRIPT" ] && [ -n "${HOME:-}" ] && SCRIPT=$(find "$HOME/.claude/skills/exa-search/" -name "exa_search.py" 2>/dev/null | head -1)
```

If `$SCRIPT` is empty after the chain, tell the user:
```
exa_search.py not found in any fallback layer ($ARIS_CACHE_DIR/tools/, ~/.config/aris/tools/, tools/, ~/.claude/skills/exa-search/).
Ensure exa-py is installed: pip install exa-py
```
Then abort this skill (Policy A — gate).

### Step 3: Execute Search

**Standard search:**
```bash
python3 "$SCRIPT" search "QUERY" --max 10 --content highlights
```

**With filters:**
```bash
python3 "$SCRIPT" search "QUERY" --max 10 \
  --category "research paper" \
  --start-date 2025-01-01 \
  --content text --max-chars 8000
```

**Find similar pages:**
```bash
python3 "$SCRIPT" find-similar "URL" --max 5 --content highlights
```

**Get content for known URLs:**
```bash
python3 "$SCRIPT" get-contents "URL1" "URL2" --content text
```

### Step 4: Present Results

Format results as a structured table:

```
| # | Title | URL | Date | Key Content |
|---|-------|-----|------|-------------|
```

For each result:
- Show title and URL
- Show published date if available
- Show highlights, text excerpt, or summary depending on content mode
- Flag particularly relevant results

### Step 5: Offer Follow-up

After presenting results, suggest:
- **Deepen**: "I can fetch full text for any of these results"
- **Find similar**: "I can find pages similar to any result"
- **Narrow**: "I can re-search with domain/date/text filters"

## Key Rules
- Always check that `EXA_API_KEY` is set before searching
- Default to `highlights` content mode for a good balance of speed and context
- Use `category: "research paper"` when the user is clearly looking for academic content
- Use `text` content mode when the user needs full page content
- Combine with `/arxiv` or `/semantic-scholar` for comprehensive literature coverage
