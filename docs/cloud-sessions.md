# Claude Code Cloud Sessions: a reference

How Claude Code cloud sessions (claude.ai/code, "Claude Code on the web")
work, from two sources: this doc was produced *inside* a cloud session
running against this repo, so section 1 draws on direct inspection of that
container; the rest draws on the official docs, reachable from inside the
session.

Every claim below is tagged:
- **[verified]** — checked by running a command in this container; the
  command is shown.
- **[docs]** — from `https://code.claude.com/docs/en/claude-code-on-the-web`
  or `https://code.claude.com/docs/en/cloud-environments`, fetched live from
  inside this session (network access was available, so nothing here is
  written from memory).

## 1. What a cloud session is

**[docs]** A cloud session is a Claude Code session that runs on cloud
infrastructure instead of your machine — by default on Anthropic-managed
infrastructure, or your org's self-hosted environment if routed there. It
keeps running after you close your laptop, and can be started from the
browser (claude.ai/code), the mobile app, the desktop app ("Cloud" instead
of "Local"), the terminal (`claude --cloud`), or as a scheduled/triggered
routine.

**[verified]** This container confirms the shape of that description:
- `env | grep CLAUDE_CODE_REMOTE` shows `CLAUDE_CODE_REMOTE=true`,
  `CLAUDE_CODE_REMOTE_ENVIRONMENT_TYPE=cloud_default`, and
  `CLAUDE_CODE_REMOTE_SESSION_ID=cse_01FzssuJm4r2FecSWbgDgmm8` — a
  cloud-specific session ID and marker not present in a local session.
- `whoami`/`id` → the session runs as `root` (uid 0).
- `cat /etc/os-release` → `Ubuntu 24.04.4 LTS`, `uname -a` →
  `Linux vm 6.18.44-fc-v33 ... x86_64`. This matches the **[docs]** claim
  that Anthropic-hosted sessions get "a fresh VM running Ubuntu 24.04 on
  x86_64, regardless of your own OS/architecture."
- `nproc` → `4`; `free -h` → `15Gi` total RAM; `df -h /` → `30G` available
  on `/dev/vda` (252G total, 7.1G used, 30G avail). This matches the
  **[docs]** stated resource ceiling of "4 vCPUs, 16 GB RAM, 30 GB disk."
  (The 252G filesystem size vs. 30G *available* suggests the VM's root disk
  is shared/overlaid infrastructure with a per-session quota, not a
  dedicated 252G disk — the 30G "Avail" figure is the number that matches
  docs, not "Size".)
- `which check-tools && check-tools` → a pre-installed script that reports
  installed toolchain versions, exactly as **[docs]** describes ("ask Claude
  to run `check-tools`"). Output confirmed Python 3.11.15, pip, poetry
  2.3.3, uv 0.8.17, black 26.3.1, mypy 1.19.1, pytest 9.0.2, ruff 0.15.8,
  Node 22.22.2 (via nvm) plus npm/yarn/pnpm/eslint/prettier/chromedriver,
  and OpenJDK 21 — matching the **[docs]** "Installed tools" table.
- `ls /opt` → `node20 node21 node22`, `ruby-3.1.6 ruby-3.2.6 ruby-3.3.6`,
  `apache-maven-3.9.11`, `gradle-8.14.3`, `rbenv`, `pw-browsers` (a
  pre-fetched Playwright/Chromium install) — consistent with **[docs]**
  saying Node versions live at `/opt/node20`, etc.
- `which uv` → `/root/.local/bin/uv`, version `0.8.17` — the same tool this
  repo's `CLAUDE.md` says to use for dependency management. No `.venv`
  exists in the repo, consistent with its flat, un-packaged layout.

### What is and isn't shared with a local session

**[docs]** (from the "What carries over from your setup" table in
Configure cloud environments): a cloud session starts from a *fresh clone*
of the repo, so anything committed to it is available, and anything
installed/configured only on your own machine is not. Specifically:

| Carries over (committed to repo) | Does not carry over (lives on your machine) |
| --- | --- |
| Repo `CLAUDE.md` | User `~/.claude/CLAUDE.md` |
| Repo `.claude/settings.json` hooks/permissions (single-repo session) | Plugins enabled only in user settings |
| Repo `.mcp.json` MCP servers (single-repo session) | MCP servers added with `claude mcp add` at local/user scope |
| Repo `.claude/rules/`, `.claude/skills/`, `.claude/agents/`, `.claude/commands/` | User `~/.claude/skills,agents,commands/` |
| Org server-managed settings (fetched fresh from Anthropic) | Device-level MDM/managed settings files |
| — | Interactive auth like AWS SSO (no browser to complete it) |
| — | Transport env vars like `NODE_EXTRA_CA_CERTS` (the hosting environment manages the API connection itself and ignores these) |

**[verified]** This repo has no `.mcp.json` and no `.claude/` directory
(`ls .mcp.json .claude` → both "No such file or directory"), and no
user-level `~/.claude/CLAUDE.md` exists either (`ls ~/.claude` shows only
harness-internal files — hooks, skills, session state — no `CLAUDE.md`).
So the *only* project instructions loaded into this session came from the
one file `find / -maxdepth 6 -iname CLAUDE.md` located:
`/home/user/statlearn/CLAUDE.md` — i.e., the repo-committed file, cloned in
with the repository. This is a direct, concrete instance of the "carries
over because it's part of the clone" row above.

## 2. The git handoff — the only channel between local and cloud

**[docs]** From the terminal, `claude --cloud "<task>"` clones your current
directory's **GitHub remote at your current branch, not your local
checkout** — so uncommitted or unpushed local work is invisible to the
cloud session unless you push first. (Exception: if there's no git remote,
or the repo is on GitHub but the Claude GitHub App isn't installed, Claude
Code instead bundles and uploads your local repo directly, including
uncommitted changes to *tracked* files but never untracked files, and
excluding credential-shaped files like `.env`/`*.pem`/`id_rsa` — see
`CCR_FORCE_BUNDLE=1` in the docs for forcing this path.)

**[verified]** This session's own git state is entirely consistent with the
"clone from remote, not local checkout" model:
- `git remote -v` → `origin https://github.com/shyngys-aitkazinov/statlearn`
  (HTTPS, not SSH — see the proxy note below).
- `git log --oneline` → only 2 commits total (`Initial project scaffold`,
  `Add ruff, mypy and pre-commit tooling`) — this is the full history of a
  brand-new repo, cloned fresh rather than carrying any local-only state.
- `git reflog show --all` → shows the actual provisioning sequence:
  `fetch --no-progress --depth 50 origin main: storing head`, then a local
  `main` branch created from `origin/main`, then a checkout onto
  `claude/eager-ritchie-rmb608`. So the container fetched with `--depth 50`
  (a shallow-ish fetch), but because this repo only has 2 commits total,
  `git rev-parse --is-shallow-repository` reports `false` — the depth limit
  never actually truncated anything.
- `git status` → clean working tree on `claude/eager-ritchie-rmb608`, and
  `git config --list` shows `user.name=Claude`, `user.email=noreply@anthropic.com`,
  plus SSH-commit-signing config (`gpg.format=ssh`, a signing key at
  `/home/claude/.ssh/commit_signing_key.pub`, `commit.gpgsign=true`) — the
  session is set up to make signed commits under a fixed identity, not
  whatever `user.*` a local clone might have.
- `git config --list` also shows `url.https://github.com/.insteadof=git@github.com:`
  and `...=ssh://git@github.com/` — SSH-form remotes get silently rewritten
  to HTTPS. This matches **[docs]**: "SSH-form GitHub remotes are rewritten
  to HTTPS automatically unless this session has its own SSH setup."
- `env | grep GH_TOKEN` → `GH_TOKEN=proxy-injected`. This is exactly the
  placeholder **[docs]** describes: "If you set neither [`GH_TOKEN` nor
  `GITHUB_TOKEN`]... both variables read as the placeholder string
  `proxy-injected`... the proxy substitutes your real credentials on
  outbound GitHub requests." The real GitHub credential never entered this
  container.

**What is lost if work isn't pushed:** per the model above, nothing in a
cloud session's working tree survives to another session, to your laptop,
or to review, except through git. **[docs]** reinforces this from the other
direction — teleporting a cloud session back to a terminal (`--teleport`)
requires "the branch from the cloud session must have been pushed to the
remote"; an unpushed branch simply isn't teleportable. Likewise, if this
container's VM is reclaimed before a push, per **[docs]** on session
expiry: "Background work that was still running when the VM was reclaimed,
such as subagents and shell commands, isn't restored" — only pushed
commits (and the conversation transcript, stored separately) persist.

## 3. How to configure a cloud environment

**[docs]** Every cloud session runs inside a **cloud environment** — a
saved, named configuration (e.g. the default one is called "Default") that
controls three things: network access, environment variables, and an
optional setup script. It's edited from the environment selector at
claude.ai/code (there is no separate settings URL for it) or from the
desktop app's prompt box.

**Setup scripts** **[docs]**: a Bash script that runs as root on Ubuntu
24.04, once per environment, before Claude Code launches — for installing
things not already pre-installed (e.g. a `.NET` SDK, or `apt install
shellcheck`). Three constraints: it must exit zero (append `|| true` to
non-critical steps), it must finish in roughly five minutes, and it needs
network access to reach package registries. After the first successful run,
Anthropic snapshots the resulting filesystem and reuses it for later
sessions in that environment — so the script normally doesn't re-run; it
reruns only when you edit the script/allowed-hosts, or the ~7-day cache
expires. For per-session project setup that should also run locally (e.g.
`npm install`), **[docs]** recommends a `SessionStart` hook in
`.claude/settings.json` instead, gated with `if [ "$CLAUDE_CODE_REMOTE" !=
"true" ]; then exit 0; fi` so it's a no-op locally.

**[verified]** `CLAUDE_CODE_REMOTE=true` is indeed present as a plain
environment variable in this session (`env | grep CLAUDE_CODE_REMOTE`),
confirming that gating idiom would work as documented.

**Environment variables** **[docs]**: set in `.env` format (`KEY=value`,
one per line) in the environment dialog; copied once into the session's
real environment variables at startup (editing them doesn't affect already-
running sessions). Anyone who can use the environment can read them back,
so secrets shouldn't go here — instead, Pro/Max plans can attach an "API
credential" to the environment, which the network proxy injects into
matching outbound requests *without the key ever entering the session's
filesystem or environment*.

**[verified]** This session's own `env` output is a live example of exactly
that proxy-injection pattern: alongside real per-session values, it
contains `AWS_ACCESS_KEY_ID=proxy-injected`,
`AWS_SECRET_ACCESS_KEY=proxy-injected`,
`CLOUDSDK_AUTH_ACCESS_TOKEN=proxy-injected`, and `GH_TOKEN=proxy-injected` —
placeholder strings, not real secrets, consistent with **[docs]**'s
description of credentials that "never reach Claude, the commands it runs,
or the session's environment variables" and are attached only after a
request leaves the VM.

**Network policy** **[docs]**: each environment picks one access level —
**None** (no outbound network through the session's own path), **Trusted**
(the default: package registries, GitHub, cloud SDKs — a large documented
allowlist), **Full** (any domain), or **Custom** (your own allowlist,
optionally plus the Trusted defaults). Regardless of level, GitHub traffic,
enabled MCP connector traffic, API-credential hosts, and the session's own
calls to `api.anthropic.com` always get through, because they route through
separate proxies rather than the session's general network path.

**[verified]** This container's network setup matches that "everything
routes through a proxy" description concretely:
- `env | grep -i proxy` → `HTTPS_PROXY=http://127.0.0.1:44617` and a large,
  explicit `no_proxy`/`NO_PROXY` allowlist whose entries
  (`registry.npmjs.org`, `pypi.org`, `files.pythonhosted.org`, `jsr.io`,
  `index.crates.io`, `proxy.golang.org`, plus `api.anthropic.com` and
  private-network ranges) line up closely with the **[docs]** "Requests that
  never get the credential" list and parts of the Trusted default allowlist.
- `cat /root/.ccr/README.md` describes the mechanism in more depth than the
  claude.ai docs do for this particular container: "Outbound HTTPS from
  this session goes through a local proxy at `http://127.0.0.1:44617`...
  which tunnels to a policy-enforcing egress proxy. TLS is re-terminated
  there, so every tool must trust the CA bundle at
  `/root/.ccr/ca-bundle.crt`." It documents fixes for TLS-trust failures,
  405s from plain-HTTP-through-HTTPS-proxy mistakes, 403/407 org-policy
  denials ("do not retry or route around it — report the blocked host"),
  and mid-transfer resets. This is a runtime detail specific to *how* the
  Trusted/Custom network policy from the claude.ai docs is enforced inside
  this particular container image, not something the public docs describe
  at this level — noted here as **[verified]** only, with no docs URL,
  since I did not find an equivalent public page for it.
- `curl` to `code.claude.com` returned HTTP `200`, confirming it's
  reachable — consistent with it being one of the "Anthropic services" in
  the **[docs]** default allowlist.

## 4. Writing a prompt for a fire-and-forget cloud session

**[docs]**, from Best practices for Claude Code, condensed to what applies
when nobody is watching the session live:

- **Give Claude something it can check itself**, or a fire-and-forget
  session just stops when the work "looks done" to it and any mistake waits
  for you to notice later. A test suite, a lint/typecheck command, a build,
  or a screenshot comparison all work. Per **[docs]**: *"write a
  validateEmail function. example test cases: user@example.com is true,
  invalid is false... run the tests after implementing"* beats *"implement a
  function that validates email addresses."*
- **Be specific about scope and constraints** — name the file, the edge
  case, the testing approach — since there's no back-and-forth to fill
  gaps mid-session.
- **Ask for evidence, not just a claim of success**: "show the test output"
  or "show the command you ran and its result" is something you can review
  later without re-running everything yourself.
- **State what's out of scope**, especially for a task you won't watch:
  Claude Code's own guidance on adversarial review suggests naming "what
  counts as a finding" and telling a reviewer to flag only things that
  "affect correctness or the stated requirements" — the same discipline
  helps a *doer* prompt stay bounded.
- **Non-interactive mode** (`claude -p "prompt"`) is the CLI's building
  block for unattended runs — the same model applies to a cloud session's
  initial task description, since after you submit it there's no
  interactive back-and-forth until you check in.
- **For anything ambiguous or architecturally significant**, expect a
  question rather than a guess: **[docs]** describes Claude's own PR
  auto-fix behavior as "if a reviewer's comment... involves something
  architecturally significant, Claude asks you before acting." A
  fire-and-forget prompt should pre-answer the questions you don't want
  asked later (branch, file layout, library choice).

## 5. Which tasks suit a cloud session (with examples from this repo)

This repo, per its own `CLAUDE.md`, is: an early-stage flat Python 3.12
script layout with no installable package, using `uv` for dependencies,
`ruff`/`mypy`/pre-commit for linting/typing, and no tests yet.

**Good fits — bounded, verifiable, don't need a human in the loop:**
- *"Add a `ruff` rule for X and fix the resulting violations across the
  repo, then run `uv run pre-commit run --all-files` and show it passing."*
  This has a hard, machine-checkable success condition
  (**[verified]** `pre-commit`, `ruff`, and `mypy` are all callable in this
  container — `which ruff` → `/root/.local/bin/ruff`, `check-tools` output
  above confirms `mypy 1.19.1`), fits inside the roughly 5-minute/4-vCPU/
  16GB budget **[docs]** states for cloud VMs, and needs zero local state.
- *"Write the first `pytest` tests for whatever `main.py` currently does,
  then run them."* The repo explicitly has none yet (`CLAUDE.md`: "No tests
  are configured yet") — a cloud session can scaffold this cleanly since
  there's no existing suite it might silently diverge from, and `pytest`
  is pre-installed (**[verified]** in `check-tools` output).
- *"Add mypy type annotations to satisfy `disallow_untyped_defs` for every
  function in the repo and confirm `uv run mypy .` is clean."* Self-checking,
  mechanical, and matches this repo's stated mypy strictness rule exactly.
- Research/reference tasks like this one — writing a doc from
  primary-source inspection plus reachable public docs — since the whole
  point is a container-isolated, disposable environment with no need to
  touch the user's machine.

**Poor fits — need a human loop, local state, or things a cloud VM lacks:**
- Anything requiring interactive OAuth/SSO — **[docs]** states this
  explicitly isn't supported ("SSO requires browser-based login that can't
  run in a cloud session"). Not relevant to this repo today, but would
  block e.g. wiring up a cloud data source that needs a login flow.
- A task that depends on **uncommitted local work**: since the session
  clones the pushed branch, not the local checkout (**[docs]**, and
  reinforced by this container's own `git log`/`git reflog` showing a fresh
  2-commit clone), any local, unpushed experiment has to be pushed first or
  it's invisible to the cloud session.
- Exploratory, open-ended design work with real back-and-forth ("what should
  the eventual package layout look like once we outgrow the flat script
  layout?") — the repo's `CLAUDE.md` flags this as a future architectural
  decision ("Adding a package later means creating `statlearn/` and
  restoring a `[build-system]` block"), which is exactly the kind of
  "architecturally significant" question **[docs]** says a session should
  stop and ask about rather than resolve unattended.
- Anything needing more than the stated **4 vCPU / 16 GB RAM / 30 GB disk**
  ceiling (**[docs]**; **[verified]** via `nproc`, `free -h`, `df -h` above)
  — not a concern for this small repo yet, but would matter the moment a
  large dataset or heavy training run enters the picture; **[docs]** points
  to Remote Control or a self-hosted environment for that case instead.
