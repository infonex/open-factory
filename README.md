# open-factory

**A software factory for AI coding agents:** Claude Code, Codex, Gemini CLI, GitHub Copilot, Cursor, or any tool
that reads [Agent Skills](https://agentskills.io). Give it a ticket, an epic, a sentence of instructions, or a queue of ready
issues. A team of agents analyses the work, writes an [OpenSpec](https://openspec.dev) spec, builds it with parallel
developers, verifies it against hidden scenarios, reviews it independently and merges it into your target branch
(`main` by default). **It never deploys or releases to production**: that stays with your own release process.

> Status: proof of concept. By default it runs as a *dark factory*: no human reads the spec or the code before it
> merges. Read [Safety](#safety) before pointing it at a production repo.

```
/factory "Add CSV export to the invoices page"
/factory ticket #142
/factory epic "Billing v2"
/factory run
```

## The flow

```mermaid
flowchart LR
    RUN(["factory run"]) --> A["Analyse"] --> S["Spec"] --> D["Develop ×N"] --> V["Verify"] --> R["Review"] --> M(["Merge to main"])
    V -. fail .-> D
    R -. fail .-> D
```

## How it works

```mermaid
flowchart TD
    IN["/factory run · ticket · epic · instructions"] --> INTAKE["Intake<br/>resolve to GitHub issues<br/>epics → dependency waves"]
    INTAKE --> AN["1 · Analyst<br/>reads issue + code<br/>scope · risk · size"]
    AN -- needs info --> BLK[("factory:blocked<br/>questions on the issue")]
    AN -- too big --> SPLIT["creates sub-issues"] --> INTAKE
    AN -- ready --> SP["2 · Spec writer<br/>OpenSpec change: proposal · specs · design · tasks<br/>work packages with file ownership<br/>+ hidden holdout scenarios"]
    SP --> GATE{"spec approval?<br/>(off by default)"}
    GATE -- off --> DEV
    GATE -- on --> HOLD[("factory:hold")]
    DEV["3 · Developers × N<br/>one worktree per work package<br/>test first → integrate → one PR"]
    DEV --> VR
    subgraph VR ["run in parallel on the same commit"]
        VE["4 · Verifier<br/>fresh context · runs the app<br/>public + hidden scenarios"]
        RE["5 · Reviewer<br/>fresh context · no developer notes<br/>spec coverage · bugs · security · gate tampering"]
    end
    VR -- fail --> FIX["Fix round<br/>developer gets findings<br/>(never the hidden scenarios)"]
    FIX -- "≤ max_fix_rounds" --> VR
    FIX -- rounds exhausted --> BLK
    VR -- pass --> HC{"hold category or<br/>merge: human?"}
    HC -- yes --> HOLD
    HC -- no --> RL["6 · Release<br/>update branch · archive spec · CI green<br/>squash merge into target branch (main)<br/>close issues"]
    RL --> DONE[("factory:done<br/>change is on main")]
    DONE -.-> PROD["Production release<br/>your own process, outside the factory"]
```

Your main agent session is the **orchestrator**. Subagents can't spawn subagents, so it spawns every agent itself,
each with a fresh context. It uses your tool's subagents, or separate headless runs if the tool has none. All state lives in GitHub: one `factory:<stage>` label per issue, one marker
comment per stage, and the spec on the change branch. A crashed or interrupted run picks up where it left off on
the next `/factory run`.

## The steps

| # | Step | Agent (default model) | What it does | Leaves behind | Exits early when |
| --- | --- | --- | --- | --- | --- |
| 0 | **Intake** | orchestrator | Turns the arguments into issues. Instructions become a new issue; an epic becomes its sub-issues, ordered in dependency waves; `run` takes every `factory:ready` issue plus anything in flight. | issues with labels | an issue is blocked, held or stopped |
| 1 | **Analyse** | analyst (Opus) | Reads the issue and the code, checks for duplicates, and decides ready, needs-info, split or reject. Classifies risk, size and sensitive categories (auth, payments, migrations…). Read-only. | analysis comment | needs-info → questions posted; too big → sub-issues created |
| 2 | **Spec** | spec-writer (Opus) | Writes an OpenSpec change on `factory/<id>`: proposal, delta specs (requirements, each with WHEN/THEN scenarios), design, and tasks grouped into **work packages** with file ownership and dependencies. Validates with `openspec validate --strict`. Also writes 3–8 **holdout scenarios** outside the repo that the developers never see. | `openspec/changes/<id>/`, `.factory/holdout/<id>.md` | the spec can't be made valid |
| 3 | **Build** | developer × N (Opus) | One developer per work package, each in its own git worktree, in dependency waves: tests first, then code, then lint, types, tests and build. With more than one package, an integrator merges the package branches and opens **one PR per issue**. The orchestrator picks N from the packages, capped by `max_parallel_developers`. | PR with `Closes #n` | a developer is blocked |
| 4 | **Verify** | verifier (Sonnet) | Fresh context; its job is to prove the change doesn't work. Runs every check, starts the app and exercises every public and hidden scenario for real (HTTP, CLI, browser), not by reading the code. Scores hidden scenarios as *satisfaction*. | verification comment | — (failures go to a fix round) |
| 5 | **Review** | reviewer (Opus) | Fresh context and deliberately no developer notes. Checks spec coverage (requirement → code → test), correctness, security, data safety, and **gate tampering** (skipped tests, loosened lint, edited CI). | review comment | — (blocking findings go to a fix round) |
| ↺ | **Fix round** | developer (Opus) | Gets the reviewer's findings word for word, and the verifier's failures as *violated requirement + observed behaviour*, never the hidden scenario. Then verify and review run again. | fix-round comment | `max_fix_rounds` used up → `factory:blocked` |
| 6 | **Release** | release (Sonnet) | One at a time (a merge queue): updates the branch, archives the OpenSpec change into the living specs, waits for CI, squash-merges into `git.target_branch` and checks the issues closed. **The factory stops here**: no deploy, no tag, no production release. | merged commit on the target branch, updated `openspec/specs/` | merge refused, or a hold applies |

## Install

Requirements:

- an AI coding tool that supports Agent Skills and, ideally, subagents
- `git`
- the [GitHub CLI](https://cli.github.com), signed in with write access to the repo
- the OpenSpec CLI (`npm i -g @fission-ai/openspec`)

### Pick your tool

Copy `skills/factory/` into your repo at the path your tool reads skills from:

| Tool | Copy `skills/factory/` to | How it runs the agents | `openspec init --tools` |
| --- | --- | --- | --- |
| Claude Code | `.claude/skills/factory/`, or install the plugin (below) | Subagents in their own worktrees, built in | `claude` |
| OpenAI Codex CLI | `.agents/skills/factory/` | Subagents | `codex` |
| Gemini CLI | `.gemini/skills/factory/` or `.agents/skills/factory/` | Subagents (experimental) | `gemini` |
| GitHub Copilot (CLI, VS Code, coding agent) | `.github/skills/factory/` | Custom agents | `github-copilot` |
| Cursor | `.cursor/skills/factory/` | Subagents | `cursor` |
| Anything else | Wherever it reads `SKILL.md` | Headless runs (`claude -p`, `codex exec`, `gemini -p`), one per stage | see `openspec init --help` |

Agents that can't get their own worktree from the tool create one with `git worktree add`; the skill tells them
how. Set `agents.models` in `factory.yaml` to your tool's model names.

> Built on Claude Code. The other tools are supported through the open Agent Skills and OpenSpec standards
> but haven't been tested yet. Skill paths are as of November 2026, so check your tool's docs.

**Claude Code plugin** (every repo on your machine):

```
/plugin marketplace add infonex/open-factory
/plugin install open-factory@open-factory
```

### Set up the repo

Run the skill's `init` from your tool. In Claude Code that's `/factory init`; in other tools, invoke the skill
their way or just ask "run factory init".

`init` does the following, then opens a setup PR. **You merge that one yourself.**

- writes `factory.yaml`, detecting your lint, test and build commands
- runs `openspec init`
- creates the labels
- git-ignores `.factory/`
- checks your CI and branch protection

In Claude Code the skill is `/factory`, or `/open-factory:factory` if another skill is already called `factory`.

## Commands

Shown in Claude Code's slash form. In other tools, invoke the skill their way, or say it in words, e.g. "factory
epic Billing v2".

| Command | Does |
| --- | --- |
| `/factory init` | One-time setup PR |
| `/factory run` | Works through every `factory:ready` issue and resumes in-flight ones |
| `/factory ticket <#, URL or title words>` | One issue |
| `/factory epic <#, milestone or title>` | A parent issue's sub-issues, a milestone, or an `epic:<name>` label, in dependency waves |
| `/factory "<instructions>"` | Opens an issue from the text, then builds it |
| `/factory status` | In-flight table; changes nothing |

Steer it with labels: add `factory:ready` to queue an issue, `factory:stop` to halt one before its next stage, and
`factory:approved` to release a hold.

## Configure

`factory.yaml` at the repo root ([template](skills/factory/templates/factory.yaml)):

| Key | Default | Meaning |
| --- | --- | --- |
| `git.target_branch` | `main` | The branch work starts from, PRs target, and the factory merges into. Use a non-deploying branch (e.g. `develop`) if pushes to `main` deploy to production |
| `policy.merge` | `auto` | `auto`: the release agent merges (dark factory). `human`: it stops at a ready PR |
| `policy.spec_approval` | `none` | `epics` or `always` stops for a human after the spec |
| `policy.hold_for_human` | `[]` | Categories that always stop before merge: `infrastructure`, `destructive-migration`, `payments`, `auth`, `ci-config`, `dependency-major`, `public-api` |
| `policy.max_fix_rounds` | `2` | Failed verify or review rounds before a human is asked |
| `policy.holdout_pass_rate` | `1.0` | Share of hidden scenarios that must pass |
| `policy.circuit_breaker.max_reverts_7d` | `2` | More reverted factory PRs than this in 7 days switches merging to `human` |
| `agents.max_parallel_developers` | `4` | Developers running at once, across all items |
| `agents.models.*` | Opus for the analyst, spec writer, developers and reviewer; Sonnet for the verifier and release | Per-agent model, in your tool's names |
| `commands.*` | detected | `install`, `lint`, `typecheck`, `test`, `build`, `start`, `e2e` |

## What's in the repo

```
.claude-plugin/          Claude Code plugin + marketplace manifests (optional)
skills/factory/
  SKILL.md               the orchestrator: arguments, stages, labels, resume
  roles/RULES.md         rules every agent follows
  roles/analyst.md       1 · analyse
  roles/spec-writer.md   2 · spec + holdouts
  roles/developer.md     3 · build (single, package, integrate, fix)
  roles/verifier.md      4 · verify
  roles/reviewer.md      5 · review
  roles/release.md       6 · merge into the target branch
  templates/factory.yaml config template
```

## How it compares

"Software factory" is the 2026 name for agent pipelines that take a spec to merged code, with other agents and
automated tests checking each step. Dan Shapiro's [five levels](https://simonwillison.net/2026/Jan/28/the-five-levels/)
put the *dark factory* at level 5: no human writes or reviews the code.

| | open-factory | StrongDM software factory | Stripe Minions | Typical published pipeline |
| --- | --- | --- | --- | --- |
| Spec | OpenSpec, written by an agent | Specs and scenarios written by humans | Task given by an engineer | Spec agent; human approves the spec |
| Hidden tests | Holdout scenarios outside the repo, scored as satisfaction | Scenarios kept as a holdout set, scored as "satisfaction" | — | Holdout scenarios outside the repo |
| Outside services | Your test environment | Digital Twin Universe: behavioural clones of Okta, Slack, Jira, Google apps | Devboxes | Sandboxes |
| Parallel builders | Yes, per work package | Yes | Yes, 1,000+ merged PRs a week | Yes |
| Code review | Independent agent | None, by rule | An engineer reviews every PR | Agents, plus humans by change class |
| Merge | Agent (configurable) | Agent | Human | Merge queue; humans for high-risk classes |
| After merge | Stops at the merge; circuit breaker on reverts | — | — | Canary, auto-rollback, monitoring agent |

The pattern open-factory takes from these: keep the code that builds and the checks that judge it separate (fresh
contexts, hidden scenarios, no developer notes for the reviewer), cap retries and escalate, never let an agent
weaken its own gates, and keep merging separate from releasing.

Sources: [StrongDM's software factory (Simon Willison)](https://simonwillison.net/2026/Feb/7/software-factory/) ·
[factory.strongdm.ai](https://factory.strongdm.ai/) ·
[Agentic software factories: issue to production](https://fdeinterviews.com/blog/agentic-software-factory) ·
[What is an AI software factory](https://futureagi.com/blog/ai-software-factory/) ·
[OpenSpec workflow](https://openspec.dev/docs/opsx)

## Safety

A dark factory ships what its checks let through. Before you use it on a production repo:

- **Merging isn't releasing.** The factory stops at the merge into `git.target_branch`. If your hosting deploys
  every push to `main` to production, point `git.target_branch` at a branch that doesn't deploy (e.g. `develop`).
  `/factory init` warns you when it spots this.
- **Your checks are the reviewer.** Require CI status checks on the target branch. Without CI, it merges on the
  verifier's local evidence alone.
- **Hold what you can't undo.** Consider `hold_for_human: [destructive-migration, infrastructure, payments, auth]`.
  A code revert doesn't bring back dropped data.
- **Issues are untrusted input.** `run` only takes issues labelled `factory:ready`, and only people with triage
  rights can add labels. Every agent treats issue and comment text as data, not instructions. Keep it that way on
  public repos.
- **Worktrees aren't sandboxes.** Agents run with your shell's permissions, network and credentials. For stronger
  isolation, run Claude Code in a container or VM with scoped tokens.
- **Holdouts are hidden by convention.** They sit in a git-ignored folder that's never passed to developers, but a
  determined agent could find it. Good enough for a proof of concept; for real isolation, store them outside the
  machine.
- **Cost.** Every stage is a fresh agent, and Opus developers in parallel add up. Watch your usage on the first
  runs.
- **Approving your own PR.** GitHub doesn't let an account approve its own PR. For `merge: auto`, require status
  checks rather than reviews, or give the factory its own bot account.

## Roadmap

- Scheduled headless mode: one CLI run per stage (`claude -p`, `codex exec`, `gemini -p`) with a spend cap per run
- Jira and Linear intake
- Digital twins of third-party services for the verifier

## License

MIT
