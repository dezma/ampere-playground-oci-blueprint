# Ampere Demo and Optimized LLM Blueprints V1

This project contains two OCI AI Blueprints payloads for a fresh OCI AI Blueprints installation.

## Files

```text
blueprints/01-demo-lite-v1.json
blueprints/02-ampere-optimized-llm-v1.json
```

## What the blueprints deploy

### 01-demo-lite-v1.json

Creates a new `VM.Standard.A2.Flex` Ampere shared pool and deploys:

- YOLOv11 demo
- Whisper demo
- Ampere Optimized Ollama
- Open WebUI connected to Ollama

Default model:

```text
llama3.2:1b
```

This profile avoids PVCs for faster first-run testing.

### 02-ampere-optimized-llm-v1.json

Creates a separate new `VM.Standard.A2.Flex` Ampere shared pool and deploys:

- Ampere Optimized Ollama
- Open WebUI connected to Ollama
- PVC-backed Ollama model cache
- PVC-backed Open WebUI data directory

Default model:

```text
hf.co/AmpereComputing/llama-3.2-3b-instruct-gguf:Llama-3.2-3B-Instruct-Q8R16.gguf
```

## Deploy

Set your Blueprints API and token:

```bash
export BP_API="https://api.<your-blueprints-domain>"
export BP_TOKEN="<your-token>"
```

Deploy demo lite:

```bash
curl -k -X POST "$BP_API/deployment/"   -H "Authorization: Token $BP_TOKEN"   -H "Content-Type: application/json"   --data-binary @blueprints/01-demo-lite-v1.json
```

Deploy optimized LLM:

```bash
curl -k -X POST "$BP_API/deployment/"   -H "Authorization: Token $BP_TOKEN"   -H "Content-Type: application/json"   --data-binary @blueprints/02-ampere-optimized-llm-v1.json
```

Deploy only one first unless you intentionally want two new Ampere shared pools.

## Watch progress

```bash
kubectl -n default get pods -w | egrep 'ampere|yolo|whisper|ollama|llmchat'
kubectl get nodes -w
```

## Open WebUI model selector

Open WebUI is configured to call Ollama through the Blueprints service port:

```text
http://${ollama.internal_dns_name}:80
```

If the model selector is empty, verify Ollama has pulled the model:

```bash
OLLAMA_POD="$(kubectl -n default get pods -o name | grep -i ollama | head -n 1)"
kubectl -n default exec "$OLLAMA_POD" -- ollama list
```

If needed, pull the model manually:

```bash
kubectl -n default exec "$OLLAMA_POD" -- ollama pull llama3.2:1b
```

For the optimized LLM profile, pull the Ampere model manually if needed:

```bash
kubectl -n default exec "$OLLAMA_POD" -- ollama pull 'hf.co/AmpereComputing/llama-3.2-3b-instruct-gguf:Llama-3.2-3B-Instruct-Q8R16.gguf'
```

## Undeploy

Use **Undeploy All In Group** in the Blueprints portal for the group you deployed:

```text
ampere-demo-lite-v1
ampere-optimized-llm-v1
```
