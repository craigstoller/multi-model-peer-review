# peer-review — shared Claude Code skill

An independent **peer review** for specs, plans, and design docs, driven through two CLIs — OpenAI's Codex and a Google Gemini model — plus, optionally, an open-weight roster (DeepSeek, Moonshot's Kimi) on the Fireworks AI API. You finish a doc; the skill runs every available engine against it from the shell in parallel, merges their findings into one severity-ordered list tagged by which engine raised each, and applies the worthwhile ones — no copy-pasting between tools. If an engine is missing or down, the review still runs on the others **and says so**. The catches only one engine makes are the point: a flaw caught in a spec now is far cheaper than the same flaw in the code built from it later.

**It reports by default and edits only when you ask.** "Peer review this" gets you findings. "Review and fix it" gets you a revised file. Proactive reviews (below) always report only.

Editing is a normal targeted diff, not a guarded operation — it does **not** checkpoint for you, and it won't refuse on an uncommitted file, since the doc you just wrote is uncommitted by definition. It pauses only when an edit would destroy work it can't see you making (changes from before this session, or a fix needing whole-section rewrites). **Commit or stash first if the file holds work you can't afford to lose.**

It's a *personal* Claude Code skill: a single self-contained `SKILL.md`.

> **Privacy — and one caveat about what "choosing the document" protects.** A review sends the full text of your doc to OpenAI (Codex) and Google (Gemini) — and, **if `FIREWORKS_API_KEY` is set in your environment**, to Fireworks AI as well, each processed under their respective data policies. The key being set is the whole enablement switch: a key set globally for unrelated Fireworks work turns the roster on. Opt out by unsetting the key for the session or removing the Engine 3 section from your installed copy. Don't peer-review documents with secrets, credentials, or sensitive IP — or limit such docs to the one engine you trust for them.
>
> **If you use the `agy` engine, selecting the document is not the whole control.** Its required `read_file(*)` grant lets it read *any* file it asks for, not only the one you named — and it resolves a bare filename by searching the filesystem. So a review could in principle read and quote a file you didn't choose. I tried three narrower grants (`read_file(<path>\*)`, bare `read_file`, forward-slash `**` globs) and agy denied all of them; only the unrestricted wildcard works, so this cannot currently be scoped. What limits it in practice: the prompt names one absolute path, plan mode blocks writes, and command execution stays denied. If that residual isn't acceptable for a given document, use the Gemini CLI engine instead — it needs no grant.

## What's in this bundle
- `peer-review/SKILL.md` — the skill itself. Drop the whole `peer-review/` folder into your personal skills directory.
- `claude-md-snippet.md` — a few lines to append to your `~/.claude/CLAUDE.md` so Claude reaches for the skill *proactively* (without being asked). Optional but recommended.

`peer-review` supersedes the earlier single-engine `codex-review-skill`, which is retired. There's no need for a separate Codex-only skill: run this one with just the Codex engine available and you get the same single-engine review, plus an explicit note that coverage was partial.

## Prerequisites (these do NOT travel inside the file)
**At least one engine must be installed and authenticated, or every run fails.** With one, you get a usable single-engine review that says it's partial. With more, you get what the skill is actually for — different labs' models disagreeing. The `superpowers` plugin is recommended but optional.

**Shell requirements — check these first if *all* engines fail identically.** The commands run under bash (Git Bash on Windows, not PowerShell) and need GNU `timeout` (with `-k`) and `realpath`:

```bash
command -v timeout      # must print a path
command -v realpath     # must print a path
```

Linux and Git Bash have them. **Stock macOS has neither** — `brew install coreutils` and ensure the GNU versions are on `PATH`. Without them every engine fails the same way, which is a single point of failure rather than the independent degradation described below, so it doesn't look like a missing CLI.

1. **OpenAI Codex CLI — installed and logged in.** The skill only *drives* Codex; it doesn't bundle it. Without it, the Codex engine is unavailable and the skill falls back to Gemini-only — and if *neither* engine is installed, every run fails.
   ```
   npm i -g @openai/codex@latest
   codex            # run once to sign in (ChatGPT account or API key)
   ```
2. **A Gemini engine — pick one.** Neither will execute shell commands embedded in a document you review — verified on both by asking each to run one mid-review, which each refused. They differ in cost, setup effort, and read scope.

   **(a) Antigravity's `agy` — recommended if you have a Gemini subscription.** Runs on plan quota rather than API billing. Install Antigravity from [antigravity.google](https://antigravity.google), then **two one-time setup steps, both required**:

   ```
   agy            # run once interactively, sign in, and ACCEPT THE TRUST PROMPT
   ```
   Then add `"read_file(*)"` to `userSettings.globalPermissionGrants.allow` in `~/.gemini/config/config.json` (create the key if absent — it sits under `userSettings`, alongside any grants already there):

   ```json
   { "userSettings": { "globalPermissionGrants": { "allow": ["read_file(*)"] } } }
   ```

   **Know what that grants.** `read_file(*)` lets agy read *any* file it asks for, not just the document under review — a broad grant, and worth weighing against the fact that agy resolves a bare filename by searching the filesystem. It does **not** permit writes or shell execution; that stays denied.

   Skip either step and every headless call is denied. The failure is loud rather than silent, but agy's error names the wrong config file, so it costs time. Verified `authMethod=consumer` with no Cloud project billed. `SKILL.md` § *Engine 2* has the command — in particular, **never add `--dangerously-skip-permissions`**, which overrides the read-only mode.

   One post-install trap: an Antigravity auto-update has been seen (2026-08-06) to delete the on-PATH launcher directory and leave the CLI at `%LOCALAPPDATA%\agy\bin\agy.exe` without re-adding it to PATH — so if `agy` vanishes from your shell after an update, look there first and put that directory back on your PATH.

   **(b) Google Gemini CLI — recommended if you don't have a subscription, or want less setup.** No permission grants to configure, and its workspace-trust gate is bypassed headlessly by the `--skip-trust` flag already in the command; `--approval-mode plan` is enforced out of the box. You still need to install it and authenticate (below).
   ```
   npm i -g @google/gemini-cli
   ```
   **Auth — read this before you try the interactive sign-in.** On **2026-06-18 Google stopped serving Gemini CLI requests for free accounts, Google AI Pro, Google AI Ultra, and individual Code Assist users.** A consumer subscription no longer authenticates this CLI; those plans route terminal use through Antigravity (`agy`) instead. Signing in interactively gets you `IneligibleTierError`.

   What still works: a **paid API key**, or a **Code Assist Standard/Enterprise** seat. Set `GEMINI_API_KEY` (from Google AI Studio) in your environment, or configure Vertex AI (`GOOGLE_GENAI_USE_VERTEXAI=true` + ADC). Billing is pay-as-you-go on the associated Cloud project and is *not* covered by any Gemini app subscription — budget a few dollars a month at typical review volume.

   Note that [Google's own auth docs](https://google-gemini.github.io/gemini-cli/docs/get-started/authentication.html) still say AI Pro/Ultra subscribers can log in, and mention neither the June change nor Antigravity. That page is stale; the error you get is accurate.

   Either path is optional in practice: if no Gemini engine is available, the skill runs Codex-only and says so.
3. **Optional — a Fireworks AI API key for the open-weight roster.** Without it the skill runs exactly as before on Codex + Gemini; with it, two more labs join every review. Create a key in the Fireworks dashboard (fireworks.ai), then set it as `FIREWORKS_API_KEY` in your environment (Windows: user environment variable, then restart your terminal; macOS/Linux: your shell profile). Billing is pay-as-you-go per token — a typical doc review across both roster models costs cents, and the edit-loop multiplies that by rounds. No CLI to install: the engine drives the REST API with `curl` + `node` — node is already required above, and curl ships with Git Bash, macOS, and Linux.
4. **The `superpowers` plugin.** The skill delegates to `superpowers:receiving-code-review` for the "apply the findings rigorously" step. These are skill *names*, not file paths, so they survive superpowers version/folder changes — but if superpowers isn't installed they fall back to a generic principle (the skill still runs, you just lose the detailed checklist). Install it from the official Claude Code plugin marketplace:
   ```
   /plugin marketplace add anthropics/claude-plugins-official
   /plugin install superpowers@claude-plugins-official
   ```

## Install

**Mac / Linux:**
```bash
git clone https://github.com/craigstoller/multi-model-peer-review.git
cd multi-model-peer-review
mkdir -p ~/.claude/skills
cp -r peer-review ~/.claude/skills/
```

**Windows (PowerShell):**
```powershell
git clone https://github.com/craigstoller/multi-model-peer-review.git
cd multi-model-peer-review
New-Item -ItemType Directory -Force "$HOME/.claude/skills" | Out-Null
Copy-Item -Recurse -Force peer-review "$HOME/.claude/skills/"
```

**Optional — proactive triggering.** Append `claude-md-snippet.md` to `~/.claude/CLAUDE.md`. Read that file first: it makes reviews happen unasked, which means documents go to OpenAI and Google without a per-run decision. **Check it isn't already there before appending** — a second copy is harmless but confusing, and there's no uninstall beyond deleting the lines.

**To update later**, re-clone or `git pull`, then re-copy `peer-review/`. Copies don't track the repo, so fixes here don't reach an installed copy until you do.

Then **restart Claude Code** (or start a new session) so it picks up the new skill.

## Usage

Ask for it in plain language — there's no command syntax to learn:

```
Run a peer review on docs/my-spec.md
Get Codex's take on this plan
Second opinion on the design doc before I send it
```

**To have it apply the findings**, ask explicitly:

```
Review docs/my-spec.md and fix what it finds
```

This edits the file in place. It does not checkpoint for you — a dirty tree is the normal case right after writing a doc, so gating on it would stall every run. Commit or stash first if the file holds work you'd hate to lose.

**Proactive triggering** (if you installed `claude-md-snippet.md`): the skill fires on its own after Claude writes or substantially revises a spec, plan, or design doc. **It reports only — a review you didn't ask for never rewrites your file.** Note that it does send the document to OpenAI and Google without asking; the snippet instructs Claude to check with you first if the content looks sensitive, but if that trade isn't right for you, skip the snippet and invoke the skill by hand.

**What comes back** is one severity-ordered list, each finding tagged by origin:

- The report opens with a **panel line** saying which engines reviewed and which were missing — read it before weighing the tags.
- Multi-engine tags (`[Codex+Kimi]`, `[All]`) — several engines converged. **Higher priority to verify, not an auto-fix** — models sharing the same document and prompt can converge on the same wrong call.
- Single tags (`[Codex]`, `[Gemini]`, `[DeepSeek]`, `[Kimi]`) — one engine's marginal catch. Scrutinize them; don't discount them for being unconfirmed.

If you asked it to apply the findings, it then tells you what it changed, what it rejected and why, and which engine surfaced anything non-obvious.

## A worked example

From this repo's own history, because it's the clearest case of the thing working.

The skill's Gemini engine had been switched to Antigravity's `agy`, invoked with `--mode plan --dangerously-skip-permissions`. The reasoning was written down and looked sound: the first flag blocks writes, the second stops headless mode from auto-denying every tool. It had been tested — a file-write attempt was correctly refused.

A peer review of an unrelated packaging document raised this, unprompted, from **one engine only**:

> Shipping a public Claude Code skill that mandates executing shell commands unsandboxed with `--dangerously-skip-permissions` creates an extreme security hazard. If a reviewed document contains prompt injection, it can execute arbitrary local shell commands without user confirmation.

That prompted an actual test — ask the engine to run `echo pwned > file` under both flags. **It ran the command and wrote the file.** `--mode plan` does not survive `--dangerously-skip-permissions`; the earlier test had verified the wrong thing.

The fix turned out to be removing the flag entirely. Two setup steps nobody had identified were the real requirement, and with those in place the engine refuses shell execution while reviews run normally.

Worth noting what the review was actually asked about: repository naming and licensing. The security finding was volunteered, wasn't confirmed by the other engine, and contradicted reasoning that had already been written down and tested. That combination — single-engine, unsolicited, contradicting the author — is exactly what the `[Codex]` / `[Gemini]` tags exist to make you look at twice.

## Verify it loaded
In a new session, ask for a review on any markdown file. If the skill loaded, it runs every engine it finds and reports which answered.

Missing or unauthenticated engines degrade gracefully — you get the remaining engines' findings plus a note. If *all* fail, check the engine prerequisites — and if they fail *identically*, check the shell requirements above, since missing `timeout`/`realpath` takes down every engine at once.

**Treat a single-engine result as partial coverage.** The premise is that different labs' models catch different things, so a one-engine review is worth materially less than a full-panel one. The output says which happened; read that line before acting on the findings.

## Note on errors
If Codex throws `unknown variant 'priority'` or `model requires a newer version of Codex`, the standalone CLI is lagging behind the Codex desktop app (they share `~/.codex/config.toml`). Fix by updating the CLI: `npm i -g @openai/codex@latest`. The SKILL.md's Troubleshooting section covers this in full.
