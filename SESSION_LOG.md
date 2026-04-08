## Session: 2026-04-08 (afternoon → evening)

### Completed
- Opened upstream PRs against `54yyyu/zotero-mcp:main`:
  - **#204** `fix: bundle of latent bug fixes uncovered during semantic reindex` — 5 commits, 33 tests
  - **#205** `feat(gemini): support gemini-embedding-2-preview + batched embedding` — 4 commits, 35 tests
- Ran 3 reviewers in parallel before submission (Claude `code-reviewer` × 2 + Codex `gpt-5.4` cross-branch). Codex caught a real correctness gap that both Claude reviewers missed.
- Added 5th commit to PR #204: `recovered_items` stats bucket (fixes a pre-existing misclassification at `semantic_search.py:857` exposed by the `_failed_docs` scope fix)
- Added v2 test commit to PR #205: 5 new tests in `TestGeminiV2Support` covering doc/query prefix routing, query truncation, batch ordering across `GEMINI_MAX_BATCH=100` chunks, and `_is_v2()` detection
- Added query-truncation fix to branch B so it's self-contained (was Codex's catch — branch B reserved `V2_PREFIX_TOKEN_BUDGET = 20` from `max_input_tokens` but `embed_query()` never actually called `self.truncate()`, making the budget reservation meaningless)
- Cherry-picked both new improvements onto the fork branch (`fix/gemini-embedding-2-preview-bundle` is now 10 commits / 38 tests)
- Reinstalled the fork via `uv tool install` and verified the runtime
- Updated `mcp-workspace` backup files (`claude_desktop_config.json` and `global-mcp-config.json`) to reference the Python fork instead of a deleted JS package path
- Updated `mcp-workspace/CLAUDE.md` inventory entry to flag zotero as `(external, forked)` and link the linxule fork
- CLAUDE.md quality audit on this fork's `CLAUDE.md`: added Tests section, fixed stale `interpretive-orchestration` reference, marked the two-zotero-mcp section as historical-only (verified no JS install exists anywhere), linked new memo
- Deleted orphaned `mcp/patches/zotero-mcp/001-gemini-embedding-2-preview-token-limit.patch` (untracked, superseded by the fork commits)

### Key Decisions
- **2-PR split** for upstream submission instead of 6 small PRs: bundle by review mindset (latent fixes vs new feature) to reduce reviewer churn while keeping each PR self-contained.
- **Duplicate the `embed_query` truncation fix in both PR branches** so neither depends on the other being merged first. When one merges, the other rebases trivially since the diffs are identical.
- **Fix the `recovered_items` misclassification in the same PR** that exposes it, rather than disclosing as a known issue. We exposed the bug, we own the fix.
- **Don't address the 9 plaintext API keys in `mcp-workspace`** — repo is private, user explicitly opted out of the cleanup. Acknowledged tech debt only.
- **Hand-edit `global-mcp-config.json`** instead of regenerating via the documented `jq` command — the file has been hand-curated beyond what `jq` would produce (13 entries vs 5 in live `~/.claude.json`), so regeneration would have wiped 8 unrelated entries.

### Next Steps
- **Watch the PRs**: `gh pr view 204 --repo 54yyyu/zotero-mcp` and `gh pr view 205 --repo 54yyyu/zotero-mcp`. Maintainer is responsive but has 72 open issues queued ahead.
- **If one PR merges before the other**, the second needs a trivial rebase to drop the duplicate `embed_query` truncation commit.
- **Follow-up PR (after #205 lands)**: bump `semantic_search.py:810` `batch_size = 25` to `100` to unlock the full 4× speedup of the `GEMINI_MAX_BATCH = 100` chunking that currently never fires.
- **Acknowledged tech debt**: 9 plaintext API keys in `linxule/mcp-workspace`. Address only if the repo flips public or gains collaborators (rotate keys → swap literals for `${...}` placeholders → optional `git filter-repo` for history scrub).

### Open Questions
- Whether to create a memex `topics/codex-review.md` topic (currently signaling `codex-delegation` instead). Garden-tending decision, not urgent.
- Whether to bump `_process_item_batch` `batch_size` to 100 in a separate small PR or wait for #205 to merge first. Probably the latter.

### Memex
- `projects/zotero-mcp/memos/2026-04-08-fork-with-gemini-embedding-2-preview-support.md` — original fork-creation session
- `projects/zotero-mcp/memos/2026-04-08-upstream-prs-and-codex-truncation-catch.md` — this session

---

## Session: 2026-02-15 18:02

### Completed
- Ran preflight diagnostics on new machine — all tools present, clean working tree
- Diagnosed MCP connection error (`[WinError 10061]`) in `k9-sniffs-claude` project
- Root cause: Zotero desktop app was not running; `ZOTERO_LOCAL=true` requires it on `localhost:23119`
- Confirmed fix: opened Zotero, verified API responding via `curl`
- No code or config changes needed

### Key Decisions
- No changes to `.mcp.json` or zotero-mcp source — config was correct, just needed Zotero running

### Next Steps
- Ensure Zotero is launched before starting Claude Code sessions that use the zotero MCP server
- Consider adding a startup check or better error message in the MCP server for when Zotero isn't running

### Open Questions
- None
