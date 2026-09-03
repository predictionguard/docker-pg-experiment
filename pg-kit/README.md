# Prediction Guard sandbox kit for Docker Sandboxes

Run AI coding agents (OpenCode) inside a [Docker Sandbox](https://docs.docker.com/ai/sandboxes/) with [Prediction Guard](https://predictionguard.com) as the model provider — network-isolated, credential-proxied, with prompt injection and PII protection built in.

## Quick start

Register your Prediction Guard API token once:

```bash
echo "$PREDICTIONGUARD_TOKEN" | sbx secret set-custom -g \
  --host pg.yourcompany.com \
  --env PREDICTIONGUARD_TOKEN \
  --placeholder sk-pg-placeholder
```

Then run:

```bash
sbx run --kit docker.io/predictionguard/sbx-predictionguard:latest predictionguard
```

Replace `pg.yourcompany.com` with your Prediction Guard deployment URL.

## What this kit does

- **Network isolation** — outbound traffic restricted to your Prediction Guard endpoint only; default deny for everything else
- **Credential proxy** — your API key stays on the host and is never exposed inside the sandbox VM
- **Model-layer governance** — every AI call passes through Prediction Guard's prompt injection detection, PII anonymization, and toxicity policies before reaching the model

## Two gates of defense

Docker Sandbox handles the runtime layer (what the agent can reach on the host and network). Prediction Guard handles the model layer (what content goes in and out of the AI). Neither gate alone is sufficient — together they enforce least-privilege at both layers.

Read the full writeup: [Two Gates of Defense — predictionguard.com/blog](https://predictionguard.com/blog/two-gates-of-defense-running-ai-agents-safely-with-docker-sandbox-and-prediction-guard)

## Source

[github.com/predictionguard/docker-pg-experiment](https://github.com/predictionguard/docker-pg-experiment)
