# Two Gates of Defense: Docker Sandbox + Prediction Guard

Reference implementation for integration between [Docker Sandbox](https://docs.docker.com/ai/sandboxes/) and [Prediction Guard](https://predictionguard.com).

Read the full writeup on the combined solution: [Two Gates of Defense — predictionguard.com/blog](https://predictionguard.com/blog/two-gates-of-defense-running-ai-agents-safely-with-docker-sandbox-and-prediction-guard)

---

## The problem

Most conversations about controlling AI agents focus on the models and input/output guardrails associated with those models (e.g., preventing prompt injections or masking PII). Although the model(s) are an important part of the supply chain of an agent, they are only a small piece of a much larger puzzle. To comprehensively govern the behavior of an agent, one needs to: (1) control the local runtime environment where the agent "harness" operates; and (2) manage the full supply chain of models, MCP servers, and tools powering agent behavior; and (3) control the agent's behavior as it operates on that distributed supply chain (and potentially interacts with other agents in a fleet).

Once you deploy an AI coding agent (Hermes Agent, Open Code, or any autonomous harness) on a developer's laptop or a cloud VM, that agent runtime can reach everything on that machine (such as SSH keys, config files, database credentials, other running services, and any API endpoint on the internet). The model doesn't matter if the agent can exfiltrate data through a path that bypasses model content filters entirely.

This is the gap that a single layer of defense can't fill, and this is why Prediction Guard has partnered with Docker's DVP agentic program to provide "two gates of defense."

**Gate #1: Docker Sandbox (SBX).** This SBX "kit" isolates the local runtime layer. The agent harness runs inside a microVM sandbox with its own kernel, filesystem, and network stack. It is locked down to only those resources that are explicitly allowed (least agency). 

**Gate #2: Prediction Guard (PG).** This "control plane" manages and controls the supply chain of allowed resources (models and tools) and how the agent behaves as it operates on those resources. Every agent action (and traces of agent behavior over time) are analyzed in real time to enforce: (a) proper agent scoping to only certain MCP servers, tools within MCP servers, and model endpoints; (b) agent behavioral controls bound to the agent's identity protecting against risks such as memory poisoning, tool misuse, runaway token use, and privilege escalation; and (c) component input and output policies for how PII, injections, and other potentially harmful context is handled in API handshakes. 

---

## Architecture

```
┌──────────────────────┐  ┌──────────────────────┐  ┌──────────────────────┐
│ Docker SBX (Gate 1)  │  │ Docker SBX (Gate 1)  │  │ Docker SBX (Gate 1)  │
│ • egress: PG only    │  │ • egress: PG only    │  │ • egress: PG only    │
│ • creds: proxied     │  │ • creds: proxied     │  │ • creds: proxied     │
│ • fs: workspace only │  │ • fs: workspace only │  │ • fs: workspace only │
│ • no pkg installs    │  │ • no pkg installs    │  │ • no pkg installs    │
│ ┌──────────────────┐ │  │ ┌──────────────────┐ │  │ ┌──────────────────┐ │
│ │ Agent 1          │ │  │ │ Agent 2          │ │  │ │ Agent N          │ │
│ │ (OpenCode)       │ │  │ │ (Hermes)         │ │  │ │ (any harness)    │ │
│ └────────┬─────────┘ │  │ └────────┬─────────┘ │  │ └────────┬─────────┘ │
└──────────┼───────────┘  └──────────┼───────────┘  └──────────┼───────────┘
           │                         │                         │
           └─────────────────────────┼─────────────────────────┘
                                     │  (only reachable endpoint)
                                     ▼
┌──────────────────────────────────────────────────────────────────────────┐
│  Self-hosted Prediction Guard control plane (Gate 2)                     │
│                                                                          │
│  • Model endpoint, MCP server & tool scoping                             │
│  • Agent tracing bound to unique agent identities                        │
│  • Immutable audit logs                                                  │
│  • Component input/output policy enforcement (prompt injection,          │
│    toxicity, PII processing, etc.)                                       │
│  • Behavioral controls tied to agent identity (privilege                 │
│    escalation, tool misuse, memory poisoning, runaway token use)         │
│  • Agent kill switches                                                   │
│  • AIBOM export in CycloneDX format (with supply chain risks)            │
│  • Human-in-the-loop approvals for tool access                           │
│  • AI security events with SIEM integration                              │
│                                                                          │
└───────┬───────────────────┬───────────────────┬───────────────────┬──────┘
        │                   │                   │                   │
        ▼                   ▼                   ▼                   ▼
┌──────────────┐    ┌──────────────┐    ┌──────────────┐    ┌──────────────┐
│ Model 1      │    │ Model N      │    │ MCP server 1 │    │ MCP server N │
└──────────────┘    └──────────────┘    └──────────────┘    └──────────────┘
```

In terms of the responsibility of each layer:

| Gate #1: Docker Verified Prediction Guard SBX Sandbox | Gate #2: Self-hosted Prediction Guard Control Plane |
|---|---|
| MicroVM runtime isolation (own kernel, filesystem, network stack) | — |
| Default-deny egress, with only the Prediction Guard API allowed | — |
| Host credential proxy, so the Prediction Guard API keys never enters the sandbox | — |
| Filesystem isolation to the mounted workspace only | — |
| Read-only host source via clone mode | — |
| Blocks package installs and capability expansion | — |
| Org-wide filesystem/network policy sync (Docker Business) | — |
| — | Scoping of model endpoints, MCP servers, and individual tools |
| — | Agent tracing bound to unique agent identities, immutable audit logs |
| — | Component input/ output policy enforcement (for prompt injection, toxicity blocking, PII processing, etc.) |
| — | Behavioral controls tied to agent identity (privilege escalation, tool misuse, memory poisoning, runaway token use) |
| — | Agent kill switches |
| — | AIBOM export in CycloneDX format (with identified supply chain risks) |
| — | Configure human-in-the-loop approvals for tool access |
| — | AI security events with SIEM integration |

---

## Sandbox kit

The `pg-kit/` directory is a Docker Sandbox kit that packages the Prediction Guard provider configuration for OpenCode. Apply it with:

```bash
sbx run --kit pg-kit/spec.yaml --name pg-opencode predictionguard
```

This kit:
- Injects `PREDICTIONGUARD_TOKEN` via the credential proxy (key stays on the host)
- Restricts outbound network to `pg.yourcompany.com`
- Drops the OpenCode provider config into the sandbox at startup

Before running, register your Prediction Guard API token as a sandbox secret:

```bash
echo "$PREDICTIONGUARD_TOKEN" | sbx secret set-custom -g \
  --host pg.yourcompany.com \
  --env PREDICTIONGUARD_TOKEN \
  --placeholder sk-pg-placeholder
```

And allow the Prediction Guard endpoint:

```bash
sbx policy allow network "pg.yourcompany.com"
```

Replace `pg.yourcompany.com` with your Prediction Guard deployment URL throughout.

---

## Test fixtures

`malicious-readme.md` and `malicious-payload.json` are the prompt injection test fixtures used in the blog post. They simulate a malicious file in the agent workspace containing instructions to exfiltrate credentials. Use them to verify that Prediction Guard's injection detection policy fires correctly.

```bash
# Test prompt injection detection (expects a blocked response)
curl https://pg.yourcompany.com/v1/chat/completions \
  -H "Authorization: Bearer $PREDICTIONGUARD_TOKEN" \
  -d @malicious-payload.json
```

---

## Prerequisites

- Prediction Guard deployment (self-hosted or cloud) with governance policies enabled
- Docker Sandbox CLI (`sbx`) — standalone, no Docker Desktop required

**macOS (Apple Silicon, Sonoma 14+)**
```bash
brew install docker/tap/sbx
sbx login
```

**Linux / cloud VM**
```bash
curl -fsSL https://get.docker.com/sbx | sh
sbx login
```

---

## Related

- [Prediction Guard docs](https://docs.predictionguard.com)
- [Docker Sandbox docs](https://docs.docker.com/ai/sandboxes/)
- [sbx-kits-contrib](https://github.com/docker/sbx-kits-contrib) — community sandbox kits
