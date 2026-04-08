# Zotero MCP — linxule fork

Personal fork of [54yyyu/zotero-mcp](https://github.com/54yyyu/zotero-mcp) with fixes for `gemini-embedding-2-preview` support and latent bugs uncovered during a 7,049-doc semantic index reindex on 2026-04-08.

**Upstream:** https://github.com/54yyyu/zotero-mcp (remote: `upstream`)
**This fork:** https://github.com/linxule/zotero-mcp (remote: `origin`)
**Active branch:** `fix/gemini-embedding-2-preview-bundle`
**Baseline:** upstream `v0.2.2` (2026-03-26)

## Why this fork exists

Upstream's `GeminiEmbeddingFunction` and `create_chroma_client` had enough latent + preview-model-specific bugs that a local patch file became unsustainable. Forking gives us:
- Clean commit history (one commit per logical fix, each upstream-able as a discrete PR)
- Ability to `uv tool install --from .` and pin the runtime to our state
- Upstream tracking via the `upstream` remote for future rebases
- A place to land follow-ups without wrestling patch-apply tooling

## What we changed vs upstream v0.2.2

6 commits on `fix/gemini-embedding-2-preview-bundle`, 8 logical fixes. See commit messages for full rationale and empirical evidence.

| Commit | Scope | File(s) |
|---|---|---|
| `feat(gemini): support gemini-embedding-2-preview + batch embedding` | Bundles 3 interdependent fixes: model-aware `max_input_tokens`, in-prompt task prefixes for v2 models (task_type silently ignored), batched embedding up to 100/call (Gemini hard cap) | `chroma_client.py` |
| `fix(gemini): truncate queries before embedding` | `embed_query()` latent bug — ran raw query text through API with no truncation | `chroma_client.py` |
| `fix(config): merge config.json with env vars instead of replacing` | **Critical silent bug.** `create_chroma_client()` unconditionally replaced `config["embedding_config"]` with env-sourced defaults whenever any provider API key was in env. Applied symmetrically to both openai and gemini branches. | `chroma_client.py` |
| `fix(semantic_search): pass _failed_docs through to _process_item_batch` | Latent scope bug — `_failed_docs` defined in `update_database` but referenced in `_process_item_batch`, causing `NameError` on every transient ChromaDB upsert failure | `semantic_search.py` |
| `fix(chroma): propagate api_key through build_from_config rehydration` | Both `GeminiEmbeddingFunction.build_from_config` and `OpenAIEmbeddingFunction.build_from_config` dropped `api_key` | `chroma_client.py` |
| `fix(gemini): reserve v2 prefix token budget in effective max_input_tokens` | Formal correctness: extract v2 prefixes to class constants, reserve `V2_PREFIX_TOKEN_BUDGET = 20`, derive effective `max_input_tokens = 7980` so post-prefix payload is formally bounded under 8192 hard cap | `chroma_client.py` |

Total diff vs upstream: `chroma_client.py +178 -52`, `semantic_search.py +29 -5`.

## Installing this fork as the active runtime

```bash
# From anywhere
uv tool install --force --from /Users/xulelin/Documents/Apps/zotero-mcp 'zotero-mcp-server[all]'

# Verify
zotero-mcp version  # should show 0.2.2
python3 -c "from zotero_mcp.chroma_client import GeminiEmbeddingFunction as F; print(F.V2_PREFIX_TOKEN_BUDGET, F.V2_DOC_PREFIX)"
# expected: 20 'Represent this document for retrieval:\n\n'
```

After this, `uv tool upgrade zotero-mcp-server` will pull from PyPI and **wipe our fixes**. Don't run it. To refresh from the fork after new commits:

```bash
cd /Users/xulelin/Documents/Apps/zotero-mcp && git pull origin fix/gemini-embedding-2-preview-bundle
uv tool install --force --from /Users/xulelin/Documents/Apps/zotero-mcp 'zotero-mcp-server[all]'
```

## Keeping up with upstream

```bash
git fetch upstream
git log upstream/main ^HEAD --oneline     # new upstream commits since our baseline
git log HEAD ^upstream/main --oneline     # our fixes not yet upstreamed
```

When upstream ships v0.2.3+, decide between:
- **Rebase our branch on upstream/main** — preserves per-commit PR structure, may need conflict resolution in the overlapping regions (`GeminiEmbeddingFunction.__call__`, `create_chroma_client`, `_process_item_batch`)
- **Cherry-pick only still-needed commits** onto a fresh branch — use this if upstream merges some of our fixes first

## Verification commands

```bash
# Syntax + constants + instance shape
python3 -c "
from zotero_mcp.chroma_client import GeminiEmbeddingFunction as F
f2 = F(model_name='models/gemini-embedding-2-preview', api_key='dummy')
f1 = F(model_name='gemini-embedding-001', api_key='dummy')
assert F.V2_PREFIX_TOKEN_BUDGET == 20
assert F.GEMINI_MAX_BATCH == 100
assert f2.max_input_tokens == 7980, f'v2 got {f2.max_input_tokens}'
assert f1.max_input_tokens == 2000, f'v1 got {f1.max_input_tokens}'
assert f2._is_v2() is True
assert f1._is_v2() is False
print('OK')
"

# End-to-end smoke (needs GEMINI_API_KEY / GOOGLE_API_KEY in env)
python3 -c "
from zotero_mcp.chroma_client import GeminiEmbeddingFunction
f = GeminiEmbeddingFunction(model_name='models/gemini-embedding-2-preview')
# Batching
vs = f(['doc1', 'doc2', 'doc3'])
assert len(vs) == 3 and all(len(v) == 3072 for v in vs)
# Query asymmetric tuning
q = f.embed_query('test query')
assert len(q) == 3072
import math
cos = sum(a*b for a,b in zip(vs[0], q)) / (math.sqrt(sum(a*a for a in vs[0])) * math.sqrt(sum(b*b for b in q)))
assert cos < 1.0, 'doc/query asymmetric tuning missing'
print(f'smoke OK, asymmetric cos={cos:.3f}')
"
```

## Known quirks (important)

### The `GOOGLE_API_KEY` leak

On this user's shell, `GOOGLE_API_KEY` is exported from somewhere (likely another LLM toolchain — `vox`, `claude-in-chrome`, or similar). Upstream's `create_chroma_client` used the presence of ANY provider API key as a switch into "env-only mode" and silently discarded the model_name from `config.json`. **The config merge fix (Commit 3) is what makes config.json work at all in this environment.** If you ever see the wrong model getting used, check:

```bash
env | grep -iE 'GOOGLE|GEMINI|OPENAI'
```

If there's more than just `GEMINI_API_KEY`, investigate.

### `gemini-embedding-2-preview` does NOT support `task_type`

The API silently ignores it. Returns bit-identical vectors to a call with no config. Verified via cosine comparison (cos=1.000000 across variants). Google's recommendation is to put the task instruction in the prompt text instead. Our fix uses two class constants:

```python
V2_DOC_PREFIX = "Represent this document for retrieval:\n\n"
V2_QUERY_PREFIX = "Represent this query for retrieval:\n\n"
```

If you change these, also update `V2_PREFIX_TOKEN_BUDGET` in lockstep — the comment on the constant flags this.

### Batch size mismatch between `semantic_search` and `chroma_client`

`semantic_search._process_item_batch` uses `batch_size = 25` (line 810). Our `GeminiEmbeddingFunction.__call__` chunks at `GEMINI_MAX_BATCH = 100`. **Net effect: `__call__` always sees at most 25 items, so the multi-chunk loop never actually fires in the reindex pipeline.** Batching is still a big win (25 sequential calls collapsed into 1 batched call), but bumping `semantic_search.py:810` to 100 would unlock the full 4× on top. Worth doing as a small follow-up PR.

### ChromaDB compaction errors during rapid reindex

Hit twice during recovery attempts, seemingly random (one compaction error, one disk I/O error). Our `_failed_docs` scope fix (Commit 4) makes these recoverable — transient ChromaDB errors are now collected and retried at end-of-run instead of crashing. Root cause of the ChromaDB errors themselves is unknown.

### Fix 7 (`build_from_config` api_key) is cosmetic for the persistence path

ChromaDB stores EF config via `get_config()` which returns `{model_name, base_url}` — no `api_key`. So during rehydration, `build_from_config(stored_config)` gets a dict without `api_key` regardless of our fix. The constructor falls back to env vars. Fix 7 only helps direct callers who pass a dict with `api_key` explicitly. Kept for symmetry; not load-bearing.

## User-facing config

Global: `~/.config/zotero-mcp/config.json`

```json
{
  "semantic_search": {
    "embedding_model": "gemini",
    "embedding_config": {
      "model_name": "models/gemini-embedding-2-preview",
      "api_key": "..."
    },
    "update_config": {
      "auto_update": true,
      "update_frequency": "startup",
      "last_update": "...",
      "update_days": 7
    },
    "extraction": {
      "pdf_max_pages": 10,
      "pdf_timeout": 30
    }
  }
}
```

Project-scoped `.mcp.json` in `seams` and `interpretive-orchestration` also sets `GEMINI_EMBEDDING_MODEL=models/gemini-embedding-2-preview` as a belt-and-suspenders override.

## Upstream PR strategy (deferred)

All 6 commits are structured as PR-sized units. When/if you want to upstream:

1. **PR A — gemini-embedding-2-preview support** (commit `b1a3afe` — Fixes 1+2+3 bundled, they are interdependent)
2. **PR B — embed_query truncation** (commit `26d1e38` — Fix 4)
3. **PR C — config.json precedence** (commit `48d746e` — Fix 5, applies to both openai and gemini branches)
4. **PR D — _failed_docs scope** (commit `02af54b` — Fix 6)
5. **PR E — build_from_config api_key passthrough** (commit `59f0cb9` — Fix 7)
6. **PR F — v2 prefix token budget** (commit `dca16da` — Fix 8, depends on PR A)

Each PR body should include the commit message verbatim — they're written to stand alone.

## Related

- Memex memo: `projects/zotero-mcp/memos/2026-04-08-fork-with-gemini-embedding-2-preview-support.md` — full debugging journey, root cause analysis, methodology notes
- Session context lives in mcp monorepo's global `/memory/` index
- Related deferred follow-up: `projects/mineru-mcp/memos/2026-04-08-mineru-zotero-integration-architecture.md` — the reason we're on this model upgrade path in the first place
