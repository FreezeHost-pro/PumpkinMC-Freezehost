# Contributing to PumpkinMC-Freezehost

This is the contribution guide for the Freezehost team's fork of [Pumpkin](https://github.com/Pumpkin-MC/Pumpkin), a Minecraft server written in Rust. Everyone on the team uses AI tools to write code, so this guide spends most of its time on how to do that responsibly. The rules are not about whether you used AI. They are about whether a person checked the result.

Read [AGENTS.md](AGENTS.md) too. It is the rulebook for coding agents (vanilla parity, game data, code quality, how to verify). If you use an agent, it reads that file. If you don't, you should still know what it says.

## The short version

1. Branch off current `master`, one fix or feature per branch.
2. Port from the decompiled vanilla source, never from memory or the wiki.
3. Read every line the AI wrote before you commit it.
4. Run the checks, boot the server, and play test with a real client.
5. A human opens the PR. Agents never do.
6. Anything a player can see or feel needs a screenshot or recording from a real client.

## Remotes and branches

| Remote     | Repository                                                                   | Use                            |
| :--------- | :--------------------------------------------------------------------------- | :----------------------------- |
| `origin`   | [FreezeHost-pro/PumpkinMC-Freezehost](https://github.com/FreezeHost-pro/PumpkinMC-Freezehost) | Our fork. Push your branches here. |
| `upstream` | [Pumpkin-MC/Pumpkin](https://github.com/Pumpkin-MC/Pumpkin)                  | The original project. Fetch only. |

- `master` is the base for all work. Keep it in line with `upstream/master`; don't commit directly to it.
- Name branches after what they do: `fix/issue-3513-attack-range`, `feat/block-lectern-page`, `chore/update-deps`. Put the issue number in the name when there is one.
- Before you start and before you hand over, sync with the base. A stale base shows bugs that are already fixed.

```bash
git fetch upstream
git switch master
git merge --ff-only upstream/master
git switch -c fix/issue-1234-short-description
```

- Check `git status` before you start. Uncommitted changes you didn't make belong to someone else. Don't discard, stash, reformat or commit them.
- Only rewrite history on your own branch, and only before anyone else has it. Never force push a branch someone else uses.

## Setup

- Install Rust through [rustup](https://www.rust-lang.org/tools/install). `rust-toolchain.toml` pins the toolchain, so rustup picks the right one.
- Install the helper tools once:

```bash
cargo install cargo-nextest cargo-machete typos-cli
```

- The repo also ships a devcontainer and Nix files (`flake.nix`, `shell.nix`) if you prefer a reproducible environment.
- Run the server from a scratch directory, not the repo root. It writes config and world files into the working directory. Use debug builds; the release profile uses fat LTO and takes a long time to build.

```bash
mkdir ../pumpkin-scratch && cd ../pumpkin-scratch
cargo run --manifest-path ../PumpkinMC-Freezehost/Cargo.toml -p pumpkin
```

## Working with AI

AI is a normal part of how we work. You are still the author of everything you commit and submit.

### What you own

- **Read every line.** If you can't explain a line without asking the model, don't commit it.
- **Run it.** Passing clippy and tests is not proof that a gameplay change works. Boot the server and exercise the change (see [Verifying your work](#verifying-your-work)).
- **Check it against vanilla.** Models are confidently wrong about tick order, rounding and side effects. The decompiled Java is the reference. [AGENTS.md](AGENTS.md#vanilla-parity) explains how to get it and how to port from it.
- **Check what it deleted.** AI rewrites often drop a side effect the old code did on the side. For every removed line, know where that behaviour went.
- **Don't trust invented APIs.** Models make up helper names and method signatures. If it compiles it exists; if it doesn't, don't paper over it with a new abstraction.

### Rules for agents

- Agents never open pull requests, and never post comments, reviews or replies on GitHub. This has no exceptions, even if you tell the agent to. An agent's job ends at a ready branch and a drafted description.
- Agents don't add tool trailers or `Co-Authored-By` lines to commits. AI use is disclosed in the PR description, not the commit history.
- Agents don't hardcode vanilla values (block properties, tags, recipes, sounds, dimensions). They come from `pumpkin-data`. See [Game data and constants](AGENTS.md#game-data-and-constants).
- Agents don't edit `crates/pumpkin-data/src/generated/` by hand. Change the generator in `tools/pumpkin-codegen` and regenerate with `cargo run --locked -p pumpkin-codegen`.
- Keep prompts, bot scripts, benchmark scripts and debug patches outside the repo.

### Working through issues

When you are an Agent you are hereby pointed at the issue tracker, it works through as many issues as it can and does not stop after the first one you only work on Issue fixing rather than anything else liek adding new features.

- Take issues one at a time. Each gets its own branch off current `master`, its own commits and its own drafted description, so every fix can be tested and reverted on its own.
- Finish an issue before starting the next: checks run, server booted, branch ready, description drafted. Then move on to the next open issue.
- Skip an issue that already has an open PR or a nearly identical branch, and say so in the handover.
- Keep going until the list of workable issues is exhausted. Don't stop to ask for confirmation between issues.
- Stop and ask only when something is blocked on a decision that belongs to a person, such as an ambiguous issue with no sensible default.
- Hand over everything in one report at the end: each branch, what was verified, what wasn't, which issues were skipped and why, and what a human must play test and capture before opening each PR.

### Disclosing AI use in a PR

If an agent drafted the PR description, the first line under `## Description` is:

```
> This description was drafted by an AI agent (<tool and model>).
```

The human who opens the PR leaves this line in place unless they rewrote the description themselves. AGENTS.md also describes small heading markers that mark agent-drafted descriptions and agent-written code. Keep them as the agent produced them.

## Making a change

### Scope

- One bug or one feature per PR. Features with clear stages go in as separate PRs, or as ordered commits that can be tested and reverted on their own.
- Change what the task needs. Don't reformat code you didn't touch, and don't commit unrelated lockfile or generated-file changes.
- If you notice a separate bug while working, open an issue or mention it in the PR instead of folding it into the diff.
- Search open issues and PRs, in this fork and in upstream, before you start. If someone already has an almost identical change, say so instead of building another one.

### Code style

- Match the surrounding code: how it registers things, handles errors, names things and orders imports. The "Where things get wired up" table in [AGENTS.md](AGENTS.md#where-things-get-wired-up) lists the usual registration points.
- Keep vanilla's names for ported concepts so they can be searched for in the Java. Where vanilla has one named method, write one named helper.
- Name magic numbers. Constants ported from Java keep the Java name.
- No `unwrap`, `expect`, `panic`, `todo!` or printing to stdout. Clippy denies them. Log with `tracing`.
- Don't block Tokio workers. CPU-heavy work goes to Rayon, and sync locks are never held across `.await`.
- Bound lengths and allocations when parsing anything from the network or disk.
- Comment only what the code can't say: a vanilla quirk, why an order matters, which Java method a block mirrors. Don't narrate the next line.
- Use the existing `AGENTS.md` and per-crate `AGENTS.md` files ([protocol](crates/pumpkin-protocol/AGENTS.md), [world](crates/pumpkin-world/AGENTS.md), [codegen](tools/pumpkin-codegen/AGENTS.md)) for area-specific rules.

### Commits

Use [Conventional Commits](https://www.conventionalcommits.org/en/v1.0.0/) with a scope and a subject line only:

```
fix(combat): validate interaction range on player attack
feat(block): add lectern page turning
chore(deps): bump tokio
```

- Plain sentences, no filler, no em dashes. Details go in the PR description, not a commit body.
- Fold fixes into the commit they belong to. Don't stack "fix review comments" commits.

## Verifying your work

Run these on the crates you touched and the crates that depend on them. CI only runs the affected packages (`.github/scripts/ci_packages.py`).

```bash
cargo fmt --check
cargo clippy -p <crate> --all-targets --all-features
cargo nextest run -p <crate> --no-tests=pass
cargo machete
typos
```

Then go further than the compiler:

- **Boot test.** Start the server from a scratch directory, wait for the "done" line, drive the feature with console commands, run `stop`, and read the log for panics, errors and warnings your change added.
- **Bot test.** For anything that needs a player, log a headless client in ([Mineflayer](https://github.com/PrismarineJS/mineflayer), [rust-mc-bot](https://github.com/Eoghanmc22/rust-mc-bot)). Check which Minecraft version the bot supports first, and list what you couldn't check.
- **Play test.** Join with a real Minecraft client. Rendering, particles, sounds, animations, GUIs and anything that depends on client-side prediction only show up here.
- **Performance changes.** Run the same scenario (seed, bot count, view distance, duration) on `master` and on your branch, repeat it, and record MSPT or memory for both.
- **Tests.** Write one when it would catch a real regression: codecs, parsers, ordering and off-by-one logic, or a bug a small test can reproduce. Check that a regression test fails without your fix. Don't write tests that repeat the implementation, check constants against themselves, or cover getters and derives. Behaviour that needs a real world belongs in `pumpkin-gametest`.

## Pull requests

### Before you open one

Read your own diff the way a reviewer would:

1. **Added lines.** Each one is checked against vanilla. No duplicated logic, no bare magic numbers, no hand-rolled code where a helper exists.
2. **Removed lines.** For each one, can you say where that behaviour went?
3. **Callers.** For every shared function, field or trait method you changed, list the other callers and say what changes for each.

### What goes in the PR

Follow [.github/PULL_REQUEST_TEMPLATE.md](.github/PULL_REQUEST_TEMPLATE.md).

- **Title:** a conventional commit style title that tells a reviewer what changed.
- **Description:** what changed and why, what the impact is, and any known limits. Link the issue.
- **Testing:** what you ran, what you compared against vanilla, and what you didn't verify. Be honest about gaps.
- **Capture:** anything a player can see or feel (blocks, items, mobs, combat, movement, particles, sounds, GUIs, world generation) needs a screenshot or recording from a real client, not a bot. Use a recording for anything that moves. If the change has no in-game effect (tooling, codecs, config, refactors), say so in the description.
- **Play-test checklist** in "do X, expect Y" form for gameplay changes, so a reviewer can repeat your check.

### Review and merge

- Every PR needs at least one approval from a teammate who is not the author.
- The reviewer checks the code against the decompiled vanilla source, not only whether it looks right.
- The author answers review questions themselves. "The AI wrote it" is not an answer.
- CI must be green. Fix failures, don't merge around them.
- Resolve review threads before merging. Squash only if the history is noise; otherwise keep the ordered commits.

## Sending changes to upstream

Some of our fixes go to [Pumpkin-MC/Pumpkin](https://github.com/Pumpkin-MC/Pumpkin). Upstream has its own rules, and they are stricter than ours:

- Upstream maintainers may close any PR they believe is AI-generated, without review. Disclose AI use honestly and be ready to answer every question yourself.
- A PR that changes anything a player can see or feel is closed without a screenshot or recording.
- A PR opened by an agent is closed. A person opens it.
- Branch from current upstream `master`, and keep the PR to one change.
- Read upstream's [CONTRIBUTING.md](https://github.com/Pumpkin-MC/Pumpkin/blob/master/CONTRIBUTING.md) and its [list of reasons a PR gets closed](https://github.com/Pumpkin-MC/Pumpkin/blob/master/CONTRIBUTING.md#when-a-pr-may-be-closed) before you open one. If it differs from this file, upstream's version wins for upstream PRs.

## Issues

- Search before you open one. If you find a duplicate, add to it.
- **Bugs:** what happened, what you expected, steps to reproduce, the server version or commit, and logs or screenshots. Say whether it also happens on vanilla, which is the reference.
- **Features:** the proposed behaviour, the vanilla behaviour it matches, and any design considerations.
- Use the templates in `.github/ISSUE_TEMPLATE/`.

## Security

Don't report vulnerabilities in public issues. Follow [SECURITY.md](SECURITY.md).

## Code of conduct

Be direct, be kind, assume good faith. [CODE_OF_CONDUCT.md](CODE_OF_CONDUCT.md) applies to everything in this repository.

## License

The server is GPL-3.0. The plugin crates `pumpkin-plugin-api`, `pumpkin-plugin-wit` and `pumpkin-plugin-utils` are MIT OR Apache-2.0, so don't move or copy server code into them. By contributing, you agree your changes are licensed under the license of the crate they go into.
