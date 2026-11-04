# Experiments

Software I'm trying out in the homelab. Not recommendations: configs here may be minimal, insecure by design for a POC, or removed without notice. Everything outside `experiments/` is what I run on
purpose.

Objects here carry the label `homelab.jakehulberg.com/experiment: "true"`.

| What | Why |
|---|---|
| kagent | Agents as Kubernetes objects (`apps/experiments/kagent`, CRDs in `infrastructure/controllers/experiments/kagent-crds`) |
| agentgateway | Proxy for LLM, MCP and A2A traffic; holds the Anthropic key for kagent (`apps/experiments/agentgateway`, controller in `infrastructure/controllers/experiments/agentgateway`) |
