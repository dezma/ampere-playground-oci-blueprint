# Ampere Demo and Optimized LLM Blueprints V1

This project contains two OCI AI Blueprints payloads for a fresh OCI AI Blueprints installation.

These payloads do **not** install OCI AI Blueprints itself. They assume the OCI AI Blueprints application stack is already running and that you can access the Blueprints API URL.

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

## Get your OCI AI Blueprints API URL and token

You need two values before deploying these JSON payloads:

```bash
export BP_API="https://api.<your-blueprints-domain>"
export BP_TOKEN="<blueprints-api-token>"
```

The `BP_TOKEN` is an **OCI AI Blueprints API token** returned by the Blueprints `/login/` endpoint. It is not an OCI CLI API key, OCI auth token, OCIR Docker auth token, or Hugging Face token.

### 1. Find the Blueprints API URL and admin credentials

After the OCI AI Blueprints **application stack** is installed, open:

```text
OCI Console
  -> Developer Services
  -> Resource Manager
  -> Stacks
  -> your OCI AI Blueprints application stack
  -> Application Information
```

Look for:

```text
OCI AI Blueprints API URL
Admin Username
Admin Password
```

Set them locally:

```bash
export BP_API="https://api.<your-blueprints-domain>"
export BP_USER="<admin-username>"
export BP_PASS='<admin-password>'
```

If the Console does not show **Application Information**, recover the values from Resource Manager job outputs in Cloud Shell.

List stacks in the compartment:

```bash
export COMPARTMENT_OCID="<compartment-ocid>"

oci resource-manager stack list \
  --compartment-id "$COMPARTMENT_OCID" \
  --all \
  --query 'data[].{"name":"display-name","id":"id","state":"lifecycle-state"}' \
  --output table
```

Set the stack OCID for the **OCI AI Blueprints application stack**, not the first OKE/VCN cluster stack:

```bash
export APP_STACK_OCID="<blueprints-application-stack-ocid>"
```

Get the latest successful apply job:

```bash
export APP_JOB_OCID="$(
  oci resource-manager job list \
    --stack-id "$APP_STACK_OCID" \
    --lifecycle-state SUCCEEDED \
    --sort-by TIMECREATED \
    --sort-order DESC \
    --query 'data[0].id' \
    --raw-output
)"
```

Print the API URL and credentials:

```bash
oci resource-manager job-output-summary list-job-outputs \
  --job-id "$APP_JOB_OCID" \
  --all \
  --output json | jq -r '
    .. | objects
    | select(has("output-name"))
    | select((.["output-name"] // "") | test("corrino_api_url|blueprints_portal_url|corrino_admin|external_ip"))
    | "\(.["output-name"])=\(.["output-value"] // "")"
  '
```

The values normally map like this:

```text
corrino_api_url                  -> BP_API
corrino_admin_username_output    -> BP_USER
corrino_admin_nonce_output       -> BP_PASS
```

### 2. Get the Blueprints API token with curl

```bash
export BP_TOKEN="$(
  curl -k -s -X POST "$BP_API/login/" \
    -F "username=$BP_USER" \
    -F "password=$BP_PASS" | jq -r '.token'
)"

echo "$BP_TOKEN"
```

A valid response from `/login/` looks like:

```json
{
  "token": "...",
  "is_new": false
}
```

If `BP_TOKEN` prints `null` or an empty string, check `BP_API`, `BP_USER`, and `BP_PASS`.

### 3. Get the token from the API web UI instead

You can also retrieve the token without curl:

```text
1. Open https://api.<your-blueprints-domain>/login/ in a browser.
2. Submit the admin username and password.
3. Copy the token from the JSON response.
4. Use it as the value for BP_TOKEN.
```

### 4. Verify the token

```bash
curl -k -s "$BP_API/oci_shapes/" \
  -H "Authorization: Token $BP_TOKEN" | jq . >/dev/null
```

If this command succeeds, you can deploy the JSON payloads.

## Deploy

Deploy demo lite:

```bash
curl -k -X POST "$BP_API/deployment/" \
  -H "Authorization: Token $BP_TOKEN" \
  -H "Content-Type: application/json" \
  --data-binary @blueprints/01-demo-lite-v1.json
```

Deploy optimized LLM:

```bash
curl -k -X POST "$BP_API/deployment/" \
  -H "Authorization: Token $BP_TOKEN" \
  -H "Content-Type: application/json" \
  --data-binary @blueprints/02-ampere-optimized-llm-v1.json
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

Or use the API endpoint:

```bash
curl -k -X POST "$BP_API/undeploy/" \
  -H "Authorization: Token $BP_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"deployment_uuid":"<deployment-uuid>"}'
```
