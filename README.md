<p align="center">
  <img src="assets/logo.png" alt="gitfix" width="96" />
</p>

<h1 align="center">gitfix</h1>

<p align="center">
  Resolve Git merge conflicts by intent, not by line.
</p>

<p align="center">
  <em>A VS Code extension for the conflicts that aren't really conflicts.</em>
</p>

<p align="center">
  <a href="https://marketplace.visualstudio.com/items?itemName=ameyypawar.gitfix">
    <img src="https://img.shields.io/visual-studio-marketplace/v/ameyypawar.gitfix?label=VS%20Code%20Marketplace" alt="VS Code Marketplace" />
  </a>
  <a href="https://gitfix.pro">
    <img src="https://img.shields.io/badge/site-gitfix.pro-1f6f5c" alt="gitfix.pro" />
  </a>
  <a href="LICENSE">
    <img src="https://img.shields.io/badge/license-MIT-555" alt="MIT" />
  </a>
</p>

<p align="center">
  <img src="assets/demo.gif" alt="gitfix resolving a merge conflict in VS Code" />
</p>

---

## Why I built it

I lost too many afternoons to merge conflicts where both sides were obviously correct and the diff just couldn't tell. Existing tools resolve text. I wanted one that resolved the change — the thing the commit was trying to do — and showed me its work before touching my files. So I built gitfix for myself, then for everyone else stuck doing the same chore.

## What it does

- Resolves Git merge conflicts by understanding what each side was trying to change, not just which lines moved.
- Explains every resolution in plain English before you accept it.
- Never writes to your working tree without an explicit click. Refuses to touch staged or committed files on its own.
- Works in any Git repository you open in VS Code — monorepos, submodules, detached HEADs, rebases, cherry-picks.
- Stays out of the way when a conflict is trivial. Hands it to you, untouched, when it isn't sure.

## See it in action

<p align="center">
  <img src="assets/screenshot-conflict.png" alt="A conflicted file opened in gitfix" />
  <br />
  <em>A real conflict, picked up automatically when the merge fails.</em>
</p>

<p align="center">
  <img src="assets/screenshot-resolved.png" alt="gitfix proposing a resolution with an explanation" />
  <br />
  <em>Proposed resolution with a one-paragraph rationale. Accept, edit, or reject.</em>
</p>

## Install

From the VS Code Marketplace:

> [marketplace.visualstudio.com/items?itemName=ameyypawar.gitfix](https://marketplace.visualstudio.com/items?itemName=ameyypawar.gitfix)

Or from the command palette:

```
ext install ameyypawar.gitfix
```

## Quick start

1. Run your merge, rebase, or cherry-pick as normal. When it stops on a conflict, open the conflicted file.
2. Open the **gitfix** panel (Command Palette → `gitfix: Resolve conflicts in this file`).
3. Review each proposed resolution. Accept, edit, or reject. Nothing is written until you say so.

That's it. There's no project to configure, no flags, no API key on the free tier.

## FAQ

**Does it modify my repo on its own?**
No. gitfix never writes to disk, stages, or commits without an explicit action from you. Closing the panel reverts the preview.

**Does it work offline?**
The core resolution flow requires a network connection. A small set of trivial conflict types resolves locally; everything else needs the gitfix service.

**Is it free?**
Yes, for individual use, with generous limits. Paid plans unlock team features and higher volume. Pricing lives at [gitfix.pro/pricing](https://gitfix.pro/pricing).

**Does my code leave my machine?**
Only the minimum needed to resolve the specific conflict you ask about — and only when you click Resolve. Nothing is stored, nothing is used for training. Details in [Privacy](#privacy).

**Does it support Mergiraf / Mergify / [tool]?**
gitfix is the layer above text-diff tools. If you already use a syntactic merger you're happy with, gitfix sits on top for the conflicts it can't handle. They compose fine.

**Which languages does it understand?**
Anything you can put in a Git repo. Quality is highest in mainstream languages today (TypeScript, Python, Go, Rust, Java, C#, Swift, Kotlin); the rest is improving steadily.

## Privacy

When you ask gitfix to resolve a specific conflict, the conflicting hunk and the immediate surrounding context are sent to the gitfix service over TLS, used to produce a resolution, and discarded. Nothing is retained, logged beyond short-lived diagnostics, or used to train any model. Full policy: [gitfix.pro/privacy](https://gitfix.pro/privacy).

## Roadmap

- Resolve conflicts across renamed and moved files without losing intent.
- Bulk-resolve a whole merge in one review pass, file by file.
- A pre-merge mode that flags conflicts that *will* happen before you start the merge.
- Better support for lockfiles, generated code, and notebook formats.
- A CLI for CI use.

## Changelog

See [CHANGELOG.md](CHANGELOG.md).

## Support

- Issues and feature requests: [GitHub Issues](https://github.com/ameyypawar/gitfix-docs/issues)
- Security: [SECURITY.md](SECURITY.md)
- Email: [ameypawar1237@gmail.com](mailto:ameypawar1237@gmail.com)

## License

The contents of this repository (docs, templates, assets) are MIT-licensed. The gitfix extension itself is closed-source and distributed via the VS Code Marketplace.
