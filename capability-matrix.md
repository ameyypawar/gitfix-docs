# gfix capability matrix

What `gfix` can do today, over MCP, with concrete proof for each row.

The honest framing: gfix is **not** "AI auto-resolves everything." Independent research (merde.ai) puts LLM resolution of complex *semantic* conflicts around ~50%. gfix's value is the stack: a deterministic floor that never regresses, an AI layer that explains and suggests, rerere that amortizes accepted answers, and audit refs that prove what happened.

---

## MCP tools (over stdio)

| Tool | What it does | What proves it works |
|---|---|---|
| `gitfix_merge_preview` | Plan an N-way merge; list resolved + unresolved without touching the tree | Audit ref `m_2026-05-22T10-25-45Z_c2bc70` — preview call ran, classified 1 unresolved conflict, returned conflict_id `c_2a3d7` |
| `gitfix_merge_status` | Report progress of the in-flight merge (state machine: planned -> resolving -> ready_to_apply -> applied) | Same audit ref; `audit.json.decisions[0]` records the state transitions |
| `gitfix_conflict_get` | Full ours/theirs/base/target bodies for one conflict; optional AI suggestion (cached for the resolve call) | Same merge; Gemini suggestion returned `confidence: 0.10` with rationale — honest about ambiguity |
| `gitfix_conflict_resolve` | Apply one resolution. Kinds: `ours` / `theirs` / `take-target` / `mergiraf` / `ai-suggestion` / `manual` | `actor: ai-agent:claude-code` in the audit log proves Slice 3's client-info extraction works |
| `gitfix_conflict_resolve_batch` | Apply N resolutions atomically (one tool call, one log entry per conflict) | Batch tool wired in M2; exercised under integration tests |
| `gitfix_merge_apply` | Finalize: materialize resolutions, write the local merge commit. **Never pushes.** Merges into `main` require `auto_approve=true` | Both dogfood audit refs (`c2bc70`, `cebcd8`) are real 3-parent octopus merge commits produced by this tool |
| `gitfix_merge_abort` | Discard the in-flight merge, restore the tree. Clears cached AI suggestions | Wired and tested; not exercised in the dogfood logs (no aborts needed) |

---

## AI paths

| Path | When it's used | What proves it |
|---|---|---|
| MCP sampling (`sampling/createMessage`) | Host advertises sampling capability. Host's LLM, host's bill, host's privacy. | Wired in Slice 3. **Today, Claude Code does NOT advertise sampling** — for Claude Code users, BYO-key is the primary path, not the fallback. |
| BYO-key (Anthropic) | `GITFIX_BYOK=1` + `[providers.anthropic]` in keys.toml | Slice 4 |
| BYO-key (OpenAI) | `GITFIX_BYOK=1` + `[providers.openai]` in keys.toml | Slice 4 |
| BYO-key (Gemini) | `GITFIX_BYOK=1` + `[providers.gemini]` in keys.toml | Slice 4.5 (emergency add mid-dogfood, see [posts/2026-06-launch.md](posts/2026-06-launch.md)). Exercised end-to-end in M2 W1 session; `source: byok-gemini` in audit log. |
| BYO-key (Ollama, fully offline) | `[providers.ollama]` in keys.toml, Ollama daemon running locally | Slice 4 |

**Provider precedence:** Anthropic -> OpenAI -> Gemini -> Ollama unless `GITFIX_BYOK_PROVIDER` overrides. Single provider per call.

**Suggestion caching:** AI suggestions returned from `gitfix_conflict_get` (with `include_ai_suggestion: true`) are cached on the merge state. `gitfix_conflict_resolve` with `kind: ai-suggestion` requires a prior cached suggestion — gfix never implicitly re-samples on resolve. Cache survives `merge_apply`; cleared on `merge_abort`.

**Budget:** 10 fresh suggestions per merge by default; cache hits and rerere replays are free. Override via env (see [setup.md](setup.md)).

**Honest ceiling:** ~50% on hard semantic conflicts (merde.ai research). Suggestions arrive with rationale and a confidence score. Low-confidence suggestions are surfaced as low-confidence — never laundered into certainty. In the M2 W1 dogfood, Gemini returned `confidence: 0.10` on a genuinely ambiguous status-text contradiction. That's the right behavior; pretending high confidence on a coin flip would be worse than the coin flip.

---

## Cross-team rerere (the amortization layer)

Every accepted resolution (except Mergiraf-deterministic and rerere-replay-itself) is written to `refs/gitfix/rerere/<blake3-hash>` as a synthetic git commit. The hash is content-addressed over `(file_path, base_oid, ours_oid, theirs_oid)`. The next merge with the same triple auto-replays — OIDs verified to guard against hash collisions.

| Capability | What proves it |
|---|---|
| Same-repo replay | Slice 5 integration tests |
| Cross-machine replay via `git push` | M2 W2 dogfood session — audit ref `m_2026-05-23T20-09-27Z_cebcd8` records `kind: rerere_replay`, `ai_suggestions_used: 0`. Wall-clock: 5 minutes (vs M2 W1's 3 calendar days for the original AI-driven resolution). |
| Share refs across teammates | Plain `git push origin refs/gitfix/rerere/*` — no gfix-specific transport. Any git remote works (GitHub, bare, self-hosted). |

**Decision-log artifact from M2 W2:**

```json
{
  "audit_schema_version": 1,
  "gitfix_version": "0.1.0",
  "merge_id": "m_2026-05-23T20-09-27Z_cebcd8",
  "decisions": [{
    "kind": "rerere_replay",
    "ai_suggestions_used": 0
  }]
}
```

The receipts are real. Fetch them yourself on a fresh clone with `git fetch origin refs/gitfix/audit/*` after the source repo lands publicly accessible refs (during alpha, fetch instructions on request via ameyap007aaa@gmail.com).

---

## Audit refs (the provenance story)

Every applied merge writes an immutable synthetic commit at `refs/gitfix/audit/<merge_id>` whose tree holds the full final merge state as `audit.json`. Visible in `git log --all`. Authored by a fixed identity (`gitfix-audit <gitfix@local>` — Slice 6).

```sh
git show refs/gitfix/audit/<merge_id>:audit.json
git push origin refs/gitfix/audit/*  # gfix never pushes; you do
```

A teammate questioning a merge three weeks later runs `git show refs/gitfix/audit/<merge_id>:audit.json` and gets the decisions, actor, AI suggestions used, rerere replays, and full final tree state in one read.

**Concrete audit refs from the dogfood logs:**

- `m_2026-05-22T10-25-45Z_c2bc70` — first AI-driven merge. `decisions[0].kind: ai_suggestion_accepted`, `actor: ai-agent:claude-code`. Gemini suggestion accepted on a same-line `.txt` conflict.
- `m_2026-05-23T20-09-27Z_cebcd8` — cross-machine rerere replay. `decisions[0].kind: rerere_replay`, `ai_suggestions_used: 0`. Identical conflict shape to the first merge, on a fresh repo, with rerere refs fetched from a shared bare remote.

---

## Untrusted-content framing (the safety property)

`gfix`'s safety property is **not** "the model can't be jailbroken." It's "a jailbroken model can only produce file content, never actions." gfix writes the model's `proposed` bytes to a file and stores `rationale` verbatim; it never executes or interprets model output.

The red-team corpus in M2 Slice 7a built this thesis into regression tests. It also found a real wrapper-escape bug — a literal `</conflict>` token in hostile content could walk the model out of the "untrusted" wrapper. Fixed in the same slice (defang any wrapper-close sequence in content); the corpus now regression-guards it. The bug being found by the red-team work it was written to find is the credibility signal worth pointing at.

---

## What gfix isn't

- **Not a merge queue.** Mergify's lane. gfix runs on a developer's machine against arbitrary branch tuples; it doesn't gate `main` or schedule merges.
- **Not an orchestrator.** Vibe Kanban's lane — 26k stars, died being an orchestrator. gfix doesn't plan agent work, dispatch tasks, or run an agent dashboard. It resolves conflicts the agents produced and then gets out of the way.
- **Not a team product yet.** Single developer, single machine is the M2 happy path. The cross-team affordances (push the rerere refs, push the audit refs) work today, but there's no team dashboard, no permission model, no centralized policy. That comes after alpha if the product earns it.
- **Not a model.** gfix doesn't train, fine-tune, or host any model. It calls your host's model via sampling or your provider via BYO-key. The intelligence in gfix is the deterministic floor + the structure around the AI call (caching, defanging, audit), not the AI itself.
- **Not magic.** ~50% AI ceiling on hard semantic conflicts. That is the number. The strategy is to make that 50% economically fine by paying for it once and replaying forever via rerere, and by never pretending the suggestion is more reliable than it is.

If you want a merge queue, use Mergify. If you want an agent orchestrator, build one or wait. If you want a tool that resolves the conflicts your agents are already producing and leaves you with provable receipts, that's gfix.
