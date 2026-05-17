---
name: deepxiv
description: Search and progressively read open-access academic papers through DeepXiv. Use when the user wants layered paper access, section-level reading, trending papers, or DeepXiv-backed literature retrieval.
argument-hint: [query-or-paper-id]
allowed-tools: Bash(*), Read, Write
---

# DeepXiv Paper Search & Progressive Reading

Search topic or paper ID: $ARGUMENTS

## Role & Positioning

DeepXiv is the **progressive-reading** literature source:

| Skill | Best for |
|------|----------|
| `/arxiv` | Direct preprint search and PDF download |
| `/semantic-scholar` | Published venue metadata, citation counts, DOI links |
| `/deepxiv` | Layered reading: search → brief → head → section, plus trending and web search |

Use DeepXiv when you want to avoid loading full papers too early.

## Constants

- **FETCH_SCRIPT** — `tools/deepxiv_fetch.py` relative to the current project. If unavailable, fall back to the raw `deepxiv` CLI.
- **MAX_RESULTS = 10** — Default number of results to return.

> Overrides (append to arguments):
> - `/deepxiv "agent memory" - max: 5` — top 5 results
> - `/deepxiv "2409.05591" - brief` — quick paper summary
> - `/deepxiv "2409.05591" - head` — metadata + section overview
> - `/deepxiv "2409.05591" - section: Introduction` — read one section only
> - `/deepxiv "trending" - days: 14 - max: 10` — trending papers
> - `/deepxiv "karpathy" - web` — DeepXiv web search
> - `/deepxiv "258001" - sc` — Semantic Scholar metadata by ID

## Setup

DeepXiv is optional. If the CLI is not installed, tell the user:

```bash
pip install deepxiv-sdk
```

On first use, `deepxiv` auto-registers a free token and stores it in `~/.env`.

## Workflow

### Step 1: Parse Arguments

Parse `$ARGUMENTS` for:

- **Query or ID**: a paper topic, arXiv ID, or Semantic Scholar ID
- **`- max: N`**: override `MAX_RESULTS`
- **`- brief`**: fetch paper brief
- **`- head`**: fetch metadata and section map
- **`- section: NAME`**: fetch one named section
- **`- trending`** or query `trending`: fetch trending papers
- **`- days: 7|14|30`**: trending time window
- **`- web`**: run DeepXiv web search
- **`- sc`**: fetch Semantic Scholar metadata by ID

If the main argument looks like an arXiv ID and no explicit mode is given, default to `- brief`.

### Step 2: Locate the Adapter (Policy D1 — primary cascade)

Resolve `deepxiv_fetch.py` via the standard fallback chain
(`shared-references/integration-contract.md` §1). Below uses `$DX` as
the resolved path:

```bash
DX=""
# Order per integration-contract.md §1: Layer 2 > Layer 3 > Layer 4 > legacy
# (Layer 1 = active_skill_dir is exposed via the runtime preamble as a
# literal path, not via env var; bundled skills have no Layer 1.)
# Layer 2: user-customised aris install (takes precedence over cache so the
# user can override individual helpers without re-bundling)
[ -n "${HOME:-}" ] && [ -f "$HOME/.config/aris/tools/deepxiv_fetch.py" ] && DX="$HOME/.config/aris/tools/deepxiv_fetch.py"
# Layer 3: aris-code v0.4.8+ bundled cache
[ -z "$DX" ] && [ -n "${ARIS_CACHE_DIR:-}" ] && [ -f "$ARIS_CACHE_DIR/tools/deepxiv_fetch.py" ] && DX="$ARIS_CACHE_DIR/tools/deepxiv_fetch.py"
# Layer 4: project workspace
[ -z "$DX" ] && [ -f "tools/deepxiv_fetch.py" ] && DX="tools/deepxiv_fetch.py"
# Legacy: ~/.claude/skills/ layout
[ -z "$DX" ] && [ -n "${HOME:-}" ] && DX=$(find "$HOME/.claude/skills/deepxiv/" -name "deepxiv_fetch.py" 2>/dev/null | head -1)

if [ -n "$DX" ]; then
  python3 "$DX" --help
fi
```

If `$DX` is empty, the SKILL still proceeds via raw `deepxiv` CLI commands as a primary cascade fallback.

### Step 3: Execute the Minimal Command

**Search papers**

```bash
if [ -n "$DX" ]; then
  python3 "$DX" search "QUERY" --max MAX_RESULTS
fi
```

Fallback:

```bash
deepxiv search "QUERY" --limit MAX_RESULTS --format json
```

**Brief summary**

```bash
if [ -n "$DX" ]; then
  python3 "$DX" paper-brief ARXIV_ID
fi
```

Fallback:

```bash
deepxiv paper ARXIV_ID --brief --format json
```

**Section map**

```bash
if [ -n "$DX" ]; then python3 "$DX" paper-head ARXIV_ID; fi
```

Fallback:

```bash
deepxiv paper ARXIV_ID --head --format json
```

**Specific section**

```bash
if [ -n "$DX" ]; then python3 "$DX" paper-section ARXIV_ID "SECTION_NAME"; fi
```

Fallback:

```bash
deepxiv paper ARXIV_ID --section "SECTION_NAME" --format json
```

**Trending**

```bash
if [ -n "$DX" ]; then python3 "$DX" trending --days 7 --max MAX_RESULTS; fi
```

Fallback:

```bash
deepxiv trending --days 7 --limit MAX_RESULTS --output json
```

**Web search**

```bash
if [ -n "$DX" ]; then python3 "$DX" wsearch "QUERY"; fi
```

Fallback:

```bash
deepxiv wsearch "QUERY" --output json
```

**Semantic Scholar metadata**

```bash
if [ -n "$DX" ]; then python3 "$DX" sc "SEMANTIC_SCHOLAR_ID"; fi
```

Fallback:

```bash
deepxiv sc "SEMANTIC_SCHOLAR_ID" --output json
```

### Step 4: Present Results

When searching, present a compact table:

```text
| # | ID | Title | Year | Citations | Notes |
|---|----|-------|------|-----------|-------|
```

When reading a paper, show:

- title
- arXiv ID
- authors
- venue/date if available
- TLDR or abstract summary
- suggested next step: `brief` → `head` → `section`

### Step 5: Escalate Depth Only When Needed

Use this progression:

1. `search`
2. `paper-brief`
3. `paper-head`
4. `paper-section`
5. full paper only if necessary

Do not jump to full-paper reads when a brief or one section answers the question.

## Key Rules

- Prefer the adapter script over raw `deepxiv` commands when available.
- DeepXiv is optional. If unavailable, give the install command and suggest `/arxiv` or `/research-lit "topic" - sources: web`.
- Use section-level reads to save tokens.
- Treat DeepXiv as complementary to `/arxiv` and `/semantic-scholar`, not a replacement.
- If the result overlaps with a published venue paper from Semantic Scholar, keep the richer venue metadata in the final summary.
