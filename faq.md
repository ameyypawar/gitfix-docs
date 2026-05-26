# Frequently Asked Questions

Ten questions a first-time user (or skeptic) is likely to ask. Honest answers; updated as alpha learnings accumulate.

---

## Why is the source private?

`gfix` is a one-developer indie product in alpha. Closed source during the brand-formation window protects against fork-and-flip while the design surface is still moving fast. A pricing decision lands at v1.0, at which point the licensing posture is re-decided as a single block — the alpha posture isn't permanent, it's provisional.

The audit refs (`refs/gitfix/audit/<merge_id>`) are the trust-substitute for source visibility. Every merge `gfix` performs writes a synthetic git commit with the full final state as `audit.json`. You can inspect, push, and share these without involving `gfix` at all — they're plain git objects.

For environments where "closed-source binary" is a hard no (compliance, security review, regulated industry): source access on request to **ameyap007aaa@gmail.com** with a one-paragraph use case. Not a public download, but not gatekept either.

---

## Who runs the AI calls? What does it cost?

Two paths:

- **MCP sampling.** When your MCP host (Claude Code, Cursor, etc.) advertises `sampling/createMessage`, `gfix` asks the host's LLM. Your model, your bill, your privacy posture — `gfix` is just the requester. **Today (alpha.1), Claude Code does NOT advertise sampling**, so this path is dormant for Claude Code users. It works in any host that does advertise sampling.
- **BYO-key.** With `GITFIX_BYOK=1` set, `gfix` calls the provider's HTTP API directly using a key from `keys.toml`. Your key, your account, your bill. `gfix` doesn't proxy or log; the API call goes provider-direct.

There is no `gfix`-operated AI backend. There is no `gfix`-bill. The free-during-alpha pricing applies to the `gfix` binary itself; AI costs accrue against your provider account directly.

Cost in practice: a single AI suggestion on a same-line text conflict costs <$0.001 on Gemini Flash (the recommended free-tier path). Rerere replay on the same conflict shape costs $0.

---

## What if my MCP host doesn't advertise sampling?

Use BYO-key. Set `GITFIX_BYOK=1` in the MCP server env (e.g., `claude mcp add --scope user -e GITFIX_BYOK=1 -- gfix /opt/homebrew/bin/gfix mcp`) and configure a provider in `keys.toml`. The recommended provider for first-time users is Gemini Flash (free tier, no billing). See [setup.md §4](setup.md#4-configure-an-ai-provider-byo-key).

This is the current reality for Claude Code users in alpha.1 — sampling is wired but no canary host advertises it. BYO-key is not a degraded mode; it's the path that works today.

---

## How is this different from Mergify / Mergiraf / Copilot's PR conflict resolver?

- **Mergify** is a merge queue. It gates `main` and serializes PRs. Different shape: gfix runs on a developer's machine against arbitrary branch tuples and never gates anything. You could use both.
- **Mergiraf** is a structural-merge tool — a deterministic AST-merger for languages with tree-sitter grammars. gfix invokes Mergiraf as a subprocess (Mergiraf is GPL-3.0; gfix never links it). Mergiraf is the M-layer of gfix's deterministic floor. We're not competing; we're a layer above.
- **GitHub Copilot's PR conflict resolver** runs in the GitHub web UI, against PRs, with GitHub's model. Different surface (web vs local), different invocation point (PR vs branch tuple), different privacy posture (Microsoft's model vs yours). gfix is the local-first, host-agnostic, MCP-native lane.

The wedge that's unique to gfix: agentic-coding workflow as the first-class user (parallel branches from Claude Code / Cursor / Aider) + the rerere amortization layer (resolve once, replay forever, even cross-machine) + audit refs as inspectable receipts.

---

## Does gfix ever modify my repo without permission?

`gfix` writes to the working tree and `.gitfix/` when you call `gitfix_merge_apply`. It writes synthetic refs under `refs/gitfix/audit/*` and `refs/gitfix/rerere/*`. That's it.

**`gfix` never pushes to remotes.** It refuses `git push`. If you want to share rerere or audit refs, you push them yourself (`git push origin refs/gitfix/rerere/*`). This is a safety boundary, not a feature gap.

**`gfix` never auto-applies AI suggestions.** A suggestion has to be explicitly accepted via `gitfix_conflict_resolve` with `kind: "ai-suggestion"`. The agent or human driving gfix makes the call; gfix records it.

**Merges into `main` require `auto_approve=true`** as an explicit flag. The default is to refuse — guardrail against an agent fat-fingering a merge into the wrong target.

Known minor issue: `gitfix_merge_apply`'s `git add -A` sweeps up `.gitfix/state.json` into the merge commit's tree if your repo has no `.gitignore` entry for `.gitfix/`. Workaround in [setup.md §6](setup.md#6-troubleshooting). Auto-gitignore fix lands in alpha.2.

---

## Can I see what gfix decided after the fact?

Yes — audit refs.

```sh
git for-each-ref refs/gitfix/audit/ --format='%(refname)'
git show refs/gitfix/audit/<merge_id>:audit.json
```

Each merge produces one `refs/gitfix/audit/<merge_id>` synthetic commit (authored by `gitfix-audit <gitfix@local>`, a fixed identity). The tree contains `audit.json` with the full final state: decisions array (per-conflict with `kind`, `actor`, AI suggestions used, rerere replays), `merge_id`, `gitfix_version`, schema version.

These refs travel with the repo when you `git push origin refs/gitfix/audit/*`. A teammate or future-you can inspect any merge gfix performed without re-running gfix — it's plain git.

---

## How do I share rerere across my team?

Push the rerere refs to any git remote:

```sh
git push origin refs/gitfix/rerere/*
```

Your teammate adds a fetch refspec to their clone (one-time config):

```sh
git config --add remote.origin.fetch '+refs/gitfix/rerere/*:refs/gitfix/rerere/*'
git fetch origin
```

Now their `gfix` will replay your accepted resolutions on the same conflict triples. Content-addressing via blake3 hashes on `(file_path, base_oid, ours_oid, theirs_oid)` means identical conflict bytes produce identical ref names — no coordination beyond pushing the refs.

This was proven cross-machine in the M2 W2 dogfood session (5-minute wall-clock vs the original W1's 3 calendar days). See [posts/2026-06-launch.md](posts/2026-06-launch.md) for the full walkthrough.

---

## Will gfix go open-source / paid / change pricing?

Honest answer: undecided. The closed-source posture and free-during-alpha pricing are both **provisional**. A pricing and licensing decision lands at v1.0 as a single block. Possible endpoints include: open core (binary closed, parts of the stack open), permanently closed-source with a paid tier, free for individuals and paid for teams, fully open-source under a permissive license. The decision will be informed by usage data from alpha.

The 5-line EULA's clause "A formal EULA will supersede this notice before v1.0" is the load-bearing commitment: whatever the v1.0 posture is, the alpha posture won't be silently extended.

What is committed: anyone who installs an alpha binary can continue to use that specific alpha binary indefinitely under the alpha EULA, regardless of what v1.0 decides. We won't retroactively change the terms on a binary you already have.

---

## macOS won't let me run gfix — what do I do?

The alpha binary is unsigned (Apple Developer ID code signing lands at v1.0). Clear the quarantine attribute:

```sh
xattr -d com.apple.quarantine /opt/homebrew/bin/gfix
```

(Substitute your install path. `which gfix` shows it.)

If that fails (rare; happens on encrypted volumes), open **System Settings -> Privacy & Security**, find the "gfix was blocked" entry, click **Open Anyway**.

Full context and alternatives in [setup.md §2](setup.md#2-macos-unsigned-binary-workaround).

---

## What languages does the structural merge (Mergiraf) support?

Mergiraf supports any language with a tree-sitter grammar in its allowlist — at time of writing, that includes Java, TypeScript, JavaScript, Python, Rust, Go, JSON, YAML, HTML, and several others (the list grows with Mergiraf upstream). For supported languages, conflicts that are syntactically reconcilable (e.g., one branch adds `async` to a function, the other changes a parameter type) get merged structurally with no conflict marker — the AI path is never even asked.

For unsupported file types (`.txt`, `.log`, `.md`, arbitrary binary), the deterministic floor falls through to plain text merge. Same-line edits produce conflict markers; the conflict goes to the AI ceiling (or you decide manually).

Check what's installed:

```sh
mergiraf --version
mergiraf list-languages  # if your Mergiraf version supports this
```

If Mergiraf isn't installed at all, gfix gracefully falls back to text merge for everything. Mergiraf is an optional accelerant, not a requirement.

---

## Where do I report bugs?

- **Install or binary bugs** -> [github.com/ameyypawar/gfix/issues](https://github.com/ameyypawar/gfix/issues)
- **Docs typos, missing platform notes, capability matrix fixes** -> [this repo's Issues](https://github.com/ameyypawar/gfix-docs/issues)
- **Security reports** -> ameyap007aaa@gmail.com (do not file public — coordinated disclosure preferred)

For binary bugs, the most useful repro includes: `gfix --version` output, OS + arch, the exact command + the MCP tool call that misbehaved, and the audit ref (if `gitfix_merge_apply` ran) — that ref's `audit.json` is the most compact way to share what gfix saw.
