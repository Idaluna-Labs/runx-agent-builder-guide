# When an agent builder should reach for runx

**Audience:** people wiring agents into real tools (Cursor, Claude Code, Codex, custom runners) who need portable skills with bounded authority — not another prompt pack.

**Links:** [runx.ai](https://runx.ai) · [github.com/runxhq/runx](https://github.com/runxhq/runx) · agent entrypoint [runx.ai/SKILL.md](https://runx.ai/SKILL.md)

---

## The problem runx actually solves

Most agent stacks blur three different jobs into one chat transcript:

1. **Judgment** — what should happen next?
2. **Effects** — call APIs, touch repos, spend money, send messages.
3. **Proof** — what was allowed, what ran, what came back?

When those stay mixed, you get ambient trust: the model “knows” a token because it saw it, a write happens because a tool was in the prompt, and tomorrow nobody can replay why a discount draft went out.

[runx](https://runx.ai) is **not** another agent framework. The upstream README is explicit: there are no model loops or vector stores in the runtime. Runx sits **under** whatever orchestrator you already use. It admits each act under explicit authority, delivers credentials without dumping them into prompt material, supervises execution, and seals a verifiable **receipt**.

That boundary is the product. Reach for runx when your agent builder pain is “skills that compound without becoming ambient trust,” not when you only need a better system prompt.

---

## When to use it (decision checklist)

Use runx when **two or more** of these are true:

- You want **expertise as a URL** (`SKILL.md` + optional `X.yaml`) that another agent can load without rebuilding a briefing packet.
- A single business signal fans into **multiple lanes** (research → draft → approval → effect) and authority must **narrow** at each edge, not pass through.
- Consequential actions need an **approval gate** before send / spend / merge / deploy.
- You need a **replayable receipt**: inputs, scopes, grants, outputs, lineage — inspectable after the chat is gone.
- You care that credentials stay out of the model context while still being available to the effect lane.

Skip runx (for now) if you are only prototyping a single local tool call with no composition, no shared skills, and no need for proof. The hello-world path still works, but the governance model pays off once graphs and receipts matter.

---

## One concrete workflow: seal a local receipt, then graduate to a governed signal

This is the path an agent builder can run today from [the public repo](https://github.com/runxhq/runx) without inventing a platform.

### Step 1 — Install the CLI and prove the local seal path

```bash
npm i -g @runxhq/cli
# or: curl -fsSL https://runx.ai/install | sh

git clone --depth 1 https://github.com/runxhq/runx.git
cd runx
runx skill ./examples/hello-world -i message="hello, runx" --json
```

What you are verifying: a skill package runs under the runtime, and the JSON result is a `runx.skill_run.v1` object that reaches `status: "sealed"` with a `receipt_id`. Inspect with:

```bash
runx history --detail --json
```

If this fails, stop and fix install / PATH before wiring agents. Receipt sealing is the contract everything else assumes.

### Step 2 — Hand your coding agent the runx skill URL

Paste into Cursor / Claude Code / similar:

`https://runx.ai/SKILL.md`

Ask the agent to use runx for a **bounded** goal, for example:

> Use runx to classify and prepare handoffs for: “acme.com signed up 40 seats yesterday.”  
> Stop before sends, spend, merges, deploys, or publishing. Return receipts.

That matches the project’s own agent-path guidance: the agent drives the runtime; consequential lanes hold at gates; you get proof instead of a one-shot transcript.

### Step 3 — Run a catalog / first-party skill that fans out

From the CLI (after you are comfortable with hello-world):

```bash
runx skill business-ops \
  -i signal="acme.com signed up 40 seats yesterday: classify the work, prepare the governed handoffs, and preserve proof" \
  --json
```

Or other practical skills already documented upstream:

```bash
runx skill sourcey -i project=. --json
runx skill deep-research -i objective="Which launch risks should we resolve first?" --json
runx skill issue-triage \
  -i issue_url=https://github.com/runxhq/runx/issues/241 \
  -i objective="Draft the next helpful maintainer response" \
  --json
```

### Step 4 — Treat the receipt as operating memory

A sealed receipt answers: what ran, who admitted it, what scopes were granted, what happened, and whether it can be verified later. Production verification needs a trusted key; local development can use the documented local-signature allowance. Do not put raw tokens in receipts — runx’s own security guidance treats that as a red line.

### Step 5 — Graduate to publishing your own skill (when ready)

Local first:

```bash
runx registry publish ./skills/<your-skill>
```

Hosted catalog when you want shared discovery (`runx login --for publish`, then publish against `https://api.runx.ai`). Hosted publishing re-runs harnesses and stores digests — publisher declaration alone is not trust.

---

## How to structure a skill so agents stay honest

Upstream splits responsibility deliberately:

| Artifact | Owns |
| --- | --- |
| `SKILL.md` | Human/agent judgment: when to use the lane, what evidence matters, where approval is required |
| `X.yaml` | Machine-checkable runners, inputs, authority, evidence contracts |
| Package JS (optional) | Deterministic domain computation the graph cannot express cleanly |
| Runtime | HTTP, filesystem, process, credentials, packets, receipts |

Keep effects in declared lanes. Naming a write in prose must not acquire the write. That is the difference between a portable skill and a brittle prompt with tools bolted on.

Minimal shape:

```markdown
---
name: hello-world
description: Echo a first Runx message through a checked-in command.
---

# Hello World

Use this package to prove the local execution and receipt path.
```

```yaml
skill: hello-world
version: "0.1.0"
runners:
  default:
    default: true
    type: cli-tool
    command: node
    args: [run.mjs]
    inputs:
      message:
        type: string
        required: true
```

---

## Practical notes for maintainers / future contributors

- **Start at the demos**, not the architecture essay. Checked-in paths like `examples/hello-world`, `skills/business-ops`, and `examples/github-mcp-hero` produce receipts rather than screenshots.
- **Read** [docs/getting-started.md](https://github.com/runxhq/runx/blob/main/docs/getting-started.md), [docs/skill-to-graph.md](https://github.com/runxhq/runx/blob/main/docs/skill-to-graph.md), and [docs/security-authority-proof.md](https://github.com/runxhq/runx/blob/main/docs/security-authority-proof.md) before proposing large design changes.
- **Build from source** with `cargo build --manifest-path crates/Cargo.toml -p runx-cli` when working on the runtime itself; the npm CLI distributes the same Rust-owned behavior.
- **Contribute** via [CONTRIBUTING.md](https://github.com/runxhq/runx/blob/main/CONTRIBUTING.md). License is Apache-2.0.

---

## Bottom line

If you are building agents that must share skills across machines, compose work under least privilege, and leave proof behind, runx is the governed runtime beneath that work. Install the CLI, seal `examples/hello-world`, give your agent [runx.ai/SKILL.md](https://runx.ai/SKILL.md), then run a real signal through `business-ops` with gates before irreversible effects. Site: [https://runx.ai](https://runx.ai). Source: [https://github.com/runxhq/runx](https://github.com/runxhq/runx).

---

*Published by Idaluna Labs as a public support / onboarding note for agent builders evaluating runx. Not affiliated with runxhq; based on the public site and repository documentation as of 2026-09-18.*
