# Tiny Bedrock Agent — README

This repo contains a **minimal AI Agent** powered by **Amazon Bedrock**, running behind **API Gateway** and **AWS Lambda**.  
You send a `POST /agent` with a message; the agent decides whether to reply directly or call a small **tool** (e.g., `get_time`, `calc`).

---

## What you get

- **Public HTTPS endpoint** (API Gateway) → `POST /agent`
- **Lambda function** (`src/app.py`) that:
  - Calls a Bedrock **foundation model**
  - Parses a JSON tool call (if the model decides a tool is needed)
  - Executes the tool and returns the result to the client
- **Two built‑in tools**:
  - `get_time(zone?)` — returns the current time in a given zone
  - `calc(op, a, b)` — does add / sub / mul / div

> You can later extend this with live weather, DynamoDB memory (todos), or multi‑step planning.

---

## Prerequisites (one‑time)

1. **AWS Account**
2. **AWS CLI** installed and configured
   ```bash
   aws --version
   aws configure
   # enter: Access Key, Secret Key, default region (recommend: us-west-2), output: json
   ```
3. **SAM CLI** (Serverless Application Model)  
   - macOS: `brew tap aws/tap && brew install aws-sam-cli`  
   - Windows: Download installer: https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/install-sam-cli.html
4. **Python 3.11+**
   - macOS (Homebrew): `brew install python@3.11`
   - Windows: https://www.python.org/downloads/
5. **Enable model access** in **Amazon Bedrock** (in the **same region** you’ll deploy to).  
   Console → Amazon Bedrock → **Model access** → enable a text model (e.g., **Amazon Titan Text Lite**).

---

## Project layout

```
.
├─ template.yaml         # SAM template (APIGW + Lambda + Bedrock permissions)
└─ src/
   └─ app.py            # Lambda handler + tool logic
```

---

## Quick start — Deploy

> Make sure you’re in the folder that contains `template.yaml`.

```bash
sam build
sam deploy --guided
```
During `--guided`, you’ll be asked:
- **Stack Name**: `tiny-bedrock-agent` (or anything)
- **Region**: e.g., `us-west-2` (must match your Bedrock model access region)
- **Allow SAM to create roles**: `Y`
- **Function has no auth** prompt: type `y` (for a public demo endpoint)
- **Save arguments to samconfig**: `Y` (optional)

At the end, SAM prints the base **API invoke URL**.  
You can also construct it manually later (see **Get your URL** below).

---

## Configure the model (optional but recommended)

Open `template.yaml` → ensure a stable model ID in your region, e.g.:

```yaml
Parameters:
  ModelId:
    Type: String
    Default: amazon.titan-text-lite-v1
```

Re‑deploy if you changed this:
```bash
sam build && sam deploy
```

> If you use Anthropic models, the **exact** `modelId` must match your region (e.g., `anthropic.claude-3-5-sonnet-20241022-v2:0`). To list all models you can use:
> ```bash
> aws bedrock list-foundation-models --by-inference-type ON_DEMAND --region us-west-2 \
>   --query "modelSummaries[].{id:modelId,name:modelName,provider:providerName}"
> ```

---

## Get your URL (if SAM didn’t print it)

1. Find your **REST API ID** (it’s also visible in the Lambda logs or CloudFormation outputs if configured):
   ```bash
   aws cloudformation list-stack-resources --stack-name tiny-bedrock-agent \
     --query "StackResourceSummaries[?ResourceType=='AWS::ApiGateway::RestApi'].PhysicalResourceId"
   ```

2. Get your **stage name(s)**:
   ```bash
   aws apigateway get-stages --rest-api-id <RestApiId> --region <region> \
     --query "item[].stageName"
   ```

3. Construct the URL:
   ```text
   https://<RestApiId>.execute-api.<region>.amazonaws.com/<Stage>/agent
   # Example:
   https://q33ea5hk7b.execute-api.us-west-2.amazonaws.com/Prod/agent
   ```

> You can also add convenient outputs to `template.yaml`:
> ```yaml
> Outputs:
>   AgentApiUrl:
>     Description: "Invoke URL for POST /agent"
>     Value: !Sub "https://${ServerlessRestApi}.execute-api.${AWS::Region}.amazonaws.com/Prod/agent"
> ```

---

## Test the agent

Replace `URL` with **your** URL:
```bash
URL="https://<restId>.execute-api.<region>.amazonaws.com/Prod/agent"

# Normal chat
curl -i -X POST "$URL" -H "Content-Type: application/json" \
  -d '{"message":"What can you do?"}'

# Tool: get_time
curl -i -X POST "$URL" -H "Content-Type: application/json" \
  -d '{"message":"What time is it in UTC?"}'

# Tool: calc
curl -i -X POST "$URL" -H "Content-Type: application/json" \
  -d '{"message":"Use calc to multiply 13 and 7"}'
```

You should receive responses like:
```json
{"reply":"2025-10-12 22:30:05 UTC"}
```
or
```json
{"reply":91.0}
```

---

## Troubleshooting

### 1) `ValidationException: Provided model identifier is invalid`
- Your `MODEL_ID` doesn’t exist in your region.
- Fix: Choose a valid one (e.g., `amazon.titan-text-lite-v1`) or list exact Anthropic IDs in your region:
  ```bash
  aws bedrock list-foundation-models --by-inference-type ON_DEMAND --region <region> \
    --query "modelSummaries[].{id:modelId,name:modelName,provider:providerName}"
  ```
- Update `template.yaml` and re‑deploy.

### 2) 500 Internal Server Error
- Tail Lambda logs to see the real error:
  ```bash
  sam logs -n AgentFn --stack-name tiny-bedrock-agent --tail
  ```
- Common causes:
  - Missing or wrong `MODEL_ID`
  - JSON body missing `"message"`
  - Region mismatch (stack vs Bedrock access)

### 3) CORS (when calling from a browser)
- For a quick test from web apps, enable CORS on the API stage or define it in the SAM template under the API event.

### 4) Securing the endpoint
- This demo is **public**. For production, add auth:
  - **AWS_IAM** (SigV4), or
  - **Cognito User Pool** authorizer, or
  - API key + usage plans.
- You can change the event in `template.yaml` to include auth settings, then re‑deploy.

---

## How to extend (next steps)

- **Live Weather tool** using Open‑Meteo (no API key).
- **Memory with DynamoDB** (`add_todo`, `list_todos`).
- **Guardrails** for basic safety policies.
- **Multi‑step planning loop** (let model call multiple tools before final answer).
- **Knowledge Bases** (Bedrock) to answer from your PDFs/FAQ in S3.

---

## Costs

- Bedrock tokens are the primary cost. For light demos you’re usually covered by **free trial/credits**.
- Lambda + API Gateway are typically pennies for demo traffic.
- Delete when done:
  ```bash
  sam delete
  ```

---

## FAQ

**Q: Did we build a “real” agent?**  
A: Yes. An “agent” is an LLM that can **decide actions** (tool calls) and **interact with an environment**. This minimal agent decides when to call `get_time` or `calc`, runs the tool, and returns the result.

**Q: Why JSON tool calls?**  
A: It’s a simple, robust way to let the model specify actions and args. The Lambda validates and executes safely.

**Q: Can I swap models?**  
A: Yes. Update the `ModelId` parameter to any valid text model your account has access to in the chosen region.

---

Happy building! 🚀
