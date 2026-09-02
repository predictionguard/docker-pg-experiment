# Two Gates of Defense: Docker Sandbox + Prediction Guard

Reference implementation for the co-marketing integration between [Docker Sandbox](https://docs.docker.com/ai/sandboxes/) and [Prediction Guard](https://predictionguard.com).

Read the full writeup: [Two Gates of Defense — predictionguard.com/blog](https://predictionguard.com/blog/two-gates-of-defense-running-ai-agents-safely-with-docker-sandbox-and-prediction-guard)

---

## The problem

AI coding agents — OpenCode, Hermes Agent, and others — run as processes with access to everything on the host machine: SSH keys, environment files, credentials, source code across all projects. A single prompt injection can weaponize that access.

Model-layer governance (Prediction Guard) catches malicious instructions before they reach the model. But it cannot control what the agent process does on the host after a response is generated.

Runtime isolation (Docker Sandbox) restricts what the agent can reach at the OS level — filesystem paths, network endpoints, installed tools. But it cannot inspect or govern the content of AI conversations.

Neither gate alone is sufficient. Together, they enforce least-privilege at both layers.

---

## Architecture

```
[Developer machine / cloud VM]
└── Docker SBX microVM (runtime isolation)
    ├── AI agent (OpenCode or Hermes Agent)
    │   └── All AI calls → Prediction Guard control plane
    │       ├── Prompt injection detection
    │       ├── Toxicity + PII policies
    │       └── Model inventory + governance
    ├── Network policy: only pg.yourcompany.com allowed outbound
    ├── Filesystem policy: only mounted workspace directories accessible
    └── Credential proxy: API key never enters the VM
```

**Gate 1 — Docker Sandbox (runtime layer)**
- MicroVM with isolated filesystem, network, and Docker daemon
- Credential proxy intercepts outbound calls and injects the real API key at the network layer — the key never lives inside the VM
- Network policy enforces allow-list: only the Prediction Guard endpoint passes through
- Filesystem policy controls which host directories can be mounted as a workspace

**Gate 2 — Prediction Guard (model layer)**
- Prompt injection detection blocks malicious instructions before they reach the model
- Toxicity and PII policies govern every conversation
- Unified model inventory across self-hosted and cloud models
- Fully self-hosted: runs inside your infrastructure, no external service calls

---

## Sandbox kit

The `pg-kit/` directory is a Docker Sandbox kit that packages the Prediction Guard provider configuration for OpenCode. Apply it with:

```bash
sbx run --kit pg-kit/spec.yaml --name pg-opencode opencode
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

- macOS Sonoma 14+ (Apple Silicon)
- Docker Desktop
- Docker Sandbox CLI: `brew install docker/tap/sbx`
- Prediction Guard deployment (self-hosted or cloud) with governance policies enabled

---

## Related

- [Prediction Guard docs](https://docs.predictionguard.com)
- [Docker Sandbox docs](https://docs.docker.com/ai/sandboxes/)
- [sbx-kits-contrib](https://github.com/docker/sbx-kits-contrib) — community sandbox kits
