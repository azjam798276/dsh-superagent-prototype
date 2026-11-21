# 009 ARTEX

[ARTEX](https://github.com/BaaRai/ARTEX) is a self-hosted, LLM-driven penetration-testing platform. It is an optional companion to dsh-superagent: it does not change how the agent handles a repository, it adds a place to send authorized security testing and a set of security playbooks to follow.

Two integration points, and they are independent:

1. **The ARTEX MCP server.** A worker Session gets ARTEX's tools (`mcp__artex__*`), so it can commission, steer and read an engagement instead of attempting the work with local tools.
2. **The ARTEX playbooks.** ARTEX ships reusable `SKILL.md` procedures (`api-security`, `patch-diff`, `bbot-recon`, `codeaudit`, …). They become skills of the agent, so ordinary security work can be done directly when it does not need a whole engagement.

Neither point weakens a boundary. What an ARTEX deployment is authorized to do is decided by that deployment's scope rules, target gate, approval gates and budgets. Commissioning a task from a worker does not widen them; the worker's `danger-full-access` shell is a separate, pre-existing risk that [006 Deployment](006-deployment.md) already covers.

## What ARTEX adds

- **Task execution off the critical path.** A request such as "audit this API for authorization flaws" becomes an ARTEX task: a planner decomposes it, workers run real scanners, and every finding keeps the captured request and response that justify it. The worker that commissioned it keeps working, and reads results later.
- **Findings with evidence.** Findings carry severity, triage status and traffic bindings. That is the difference between "the model thinks this endpoint is broken" and a report a person can check.
- **Reusable procedures.** The playbooks in `skills/` are the same ones ARTEX's own agents follow, so a worker following `api-security` locally and an ARTEX worker doing it are reading the same instructions.

## Prerequisites

- A reachable ARTEX instance. The two-process (backend + MCP server) setup below needs ARTEX built from a checkout that includes `-mcp-serve`; see the ARTEX fork's `docs/mcp-server.md`.
- An ARTEX API token: `./scripts/mint-mcp-token.py --url http://127.0.0.1:8787 --print-env`, or `localStorage.artex_token` copied from a signed-in console. Tokens live 7 days.
- The `@deepseek-ai/dsh-mcp-client` plugin, which the Web profile already provides.

## 1. Configure the MCP server

Add the fragment in [`examples/artex.cordis.patch.yml`](../examples/artex.cordis.patch.yml) to the profile's `cordis.patch.yml`, after the bundle's own rows, and set `command`/`cwd` to your ARTEX binary.

The token does not belong in that file. It is ordinary world-readable configuration; pass it through the profile environment instead:

```bash
# systemd: EnvironmentFile=/etc/dsh/superagent.env
ARTEX_API_TOKEN=<token>
```

### Why the fragment does not edit the worker preset

Verified against `dsh --dump-config`: a patch row's `config` **replaces a whole config**, and patch ids are indexed at the top level of the composed tree. The rows inside `dsh-superagent-worker` — the preset in the bundle's `cordis.patch.yml` — are therefore not addressable from a profile patch, and a fragment that looked like it edited them would fail with `patch: entry not found`.

So the fragment inserts `mcp-client` at profile level, where it publishes its tools to every Agent composed under the profile. That covers worker Sessions without touching the preset. The cost is that the tools also appear in the main (router) Session and in personal Sessions in the same profile; if you need them only in workers, disable the row there instead of fighting the patch layering.

## 2. Install the playbooks

`skill-filesystem` reads `$HOME/.agents/skills` and `$DSH_HOME/skills` by default and follows symlinks, so the playbooks need no configuration:

```bash
mkdir -p ~/.agents/skills
for d in /opt/artex/skills/*/; do ln -sfn "$d" ~/.agents/skills/; done
```

Symlinking, rather than copying, means a playbook ARTEX updates is the one the agent reads. The fragment's commented `customSkillDirs` row is the alternative when the playbooks should stay out of the shared user root; use one or the other, not both, or every playbook is listed twice.

## 3. Verify

Ask the agent for a worker-visible check rather than trusting the configuration:

- In a Session, the `mcp__artex__artex_tasks_list` tool exists and returns the instance's tasks.
- A `401` in its result means the token is wrong or expired, not that the wiring is wrong; the tool says so in its text.
- The worker's skill list includes `api-security` and `artex-mcp`. The second is ARTEX's own brief for an agent driving the platform: read before you write, write goals that can be proven, never quote a finding as verified without checking its evidence.

## Troubleshooting

| Symptom | Cause |
| --- | --- |
| `patch: entry "mcp-artex" not found` | The fragment was added before the bundle rows, or its `insert` nesting was flattened. |
| Tools missing, health reports the server | `failOnStartupError: false` keeps the Session working while ARTEX is down; the MCP row reports the failed connection. |
| `HTTP 401` from every tool | Token expired (7 days) or minted by a different instance. |
| Connect fails immediately | `command` is not the ARTEX binary, or it predates `-mcp-serve`. Check it runs `artex -mcp-serve -h`. |
| Findings pages time out | Raise the client's `toolCallTimeoutMs`; a large engagement's page can take seconds. |

## What this does not do

- It does not give ARTEX the agent's repository or credentials. A task only knows what its `description` says and what the instance already holds in its asset graph.
- It does not let a worker read a finding as a fact. A finding is a claim with evidence attached; the `artex-mcp` skill says to open the evidence before repeating the claim.
- It does not replace the agent's own review of its work. A penetration test of a target the operator supplied and a change to the repository are different risks, and the second still needs the review [003 Execution](003-execution.md) describes.
