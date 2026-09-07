# Build, run and operate an agent on the AgentCore harness

*Hands-on lab · 12 exercises · Live AWS account*

The harness is the orchestration loop — model calls, tool selection, context management, failure handling — delivered as a managed service. You declare what the agent *is*; AWS runs the loop inside an isolated microVM. This lab takes you from an empty account to a versioned, observable, VPC-attached agent, then exports it to code.

| Region | Time | Cost | Prereqs | Cleanup |
|---|---|---|---|---|
| us-west-2 | 3–4 hours | Pay-per-use | Node 20+ · Python 3.10+ | Lab 12 |

## Contents

- [Read this first](#read-this-first)
- [01. Set up the toolchain, execution role and model access](#01-set-up-the-toolchain-execution-role-and-model-access)
- [02. Create and invoke your first harness](#02-create-and-invoke-your-first-harness)
- [03. Models and instructions](#03-models-and-instructions)
- [04. Tools](#04-tools)
- [05. Skills](#05-skills)
- [06. Memory](#06-memory)
- [07. Environment and filesystem](#07-environment-and-filesystem)
- [08. Observability and cost controls](#08-observability-and-cost-controls)
- [09. Versioning and endpoints](#09-versioning-and-endpoints)
- [10. Security and access controls](#10-security-and-access-controls)
- [11. Export the harness to code](#11-export-the-harness-to-code)
- [12. Harness or Runtime — and cleaning up](#12-harness-or-runtime-and-cleaning-up)
- [Reset the account and run the lab again](#reset-the-account-and-run-the-lab-again)
- [Reference](#reference)

## Read this first

### What you are actually building

Across twelve labs you build one agent — a research assistant — and progressively attach every harness capability to it: models you can swap mid-conversation, tools from four different sources, domain skills pulled from Git and S3, persistent per-user memory, a filesystem that outlives the microVM, execution limits, traces, immutable versions behind named endpoints, and finally an escape hatch to plain Python.

### Three ways to drive the harness

Every lab gives you the same operation in whichever interface you prefer. Pick one and stay with it, or switch freely — they act on the same resources.

| Interface | Install | Best for |
|---|---|---|
| AgentCore CLI | `npm i -g @aws/agentcore` | Fastest path. Scaffolds the IAM role and project for you. |
| AWS CLI | `aws bedrock-agentcore-control` | Seeing the raw API shape; scripting in CI. |
| boto3 | `pip install boto3` | Calling the harness from your own application. |

> **Warning — This lab creates billable resources**
>
> There is no separate charge for the harness itself — you pay for the underlying capabilities it uses: model inference, Runtime microVM time, Memory, Browser, Code Interpreter, CloudWatch. Every lab is small, but idle microVMs stay warm for 15 minutes by default. **Run Lab 12 when you are done.**

> **Note — Region and availability**
>
> The harness is GA in the AgentCore regions. This lab standardises on `us-west-2`, with one exception: the Web Search connector in Lab 04 is `us-east-1` only, and that step is marked optional.

## 01. Set up the toolchain, execution role and model access

*~25 min · IAM · model access · shell environment*

> **Goal**
>
> Establish shell variables used by every later lab, create the IAM execution role the harness assumes, and enable access to the model it will call. Everything the agent does at runtime — calling Bedrock, pulling its container, writing traces — happens under this role.

### Step 1 — Shell environment

Open a terminal and export these. Every command in this lab assumes they are set.

```bash
export AWS_REGION=us-west-2
export AWS_DEFAULT_REGION=us-west-2
export ACCOUNT_ID="$(aws sts get-caller-identity --query Account --output text)"
export ROLE_NAME=AgentCoreHarnessLabRole
export ROLE_ARN="arn:aws:iam::${ACCOUNT_ID}:role/${ROLE_NAME}"

echo "Account $ACCOUNT_ID / Region $AWS_REGION"
echo "$ROLE_ARN"
```

### Step 2 — Install a client

#### AgentCore CLI

*needs Node 20+*

```bash
npm install -g @aws/agentcore
agentcore --version
```

The CLI is the fastest path: it scaffolds a project, *creates the execution role for you*, and deploys via CloudFormation. If you use only the CLI you may skip Step 3 — but read it anyway, because Lab 10 asks you to reason about these permissions.

On a rerun, skip this step. The package is installed globally and nothing in [Reset & rerun](#reset-the-account-and-run-the-lab-again) removes it — `agentcore --version` will already answer.

#### AWS CLI + boto3

*needs Python 3.10+*

```bash
python -m pip install --upgrade boto3
python -c "import boto3; print(boto3.__version__)"
aws --version
```

Your *caller* identity also needs harness permissions. Note the dual-permission model: each harness API requires both the harness action and the underlying Runtime action — `InvokeHarness` needs `bedrock-agentcore:InvokeHarness` *and* `bedrock-agentcore:InvokeAgentRuntime`. Lab 10 has the full matrix.

### Step 3 — Create the execution role

The trust policy lets the AgentCore service principal assume the role.

*trust policy*

```bash
cat > trust-policy.json <<'JSON'
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {"Service": "bedrock-agentcore.amazonaws.com"},
    "Action": "sts:AssumeRole"
  }]
}
JSON

aws iam create-role \
  --role-name "$ROLE_NAME" \
  --assume-role-policy-document file://trust-policy.json
```

Now the permission policy. This is the baseline every harness needs: model invocation, the public ECR pull for the harness application image, X-Ray traces, CloudWatch logs and metrics, workload identity, and the default Browser and Code Interpreter tools.

*execution policy*

```bash
cat > harness-policy.json <<JSON
{
  "Version": "2012-10-17",
  "Statement": [
    { "Sid": "BedrockModelInvocation", "Effect": "Allow",
      "Action": ["bedrock:InvokeModel","bedrock:InvokeModelWithResponseStream"],
      "Resource": ["arn:aws:bedrock:*::foundation-model/*",
                   "arn:aws:bedrock:${AWS_REGION}:${ACCOUNT_ID}:*"] },
    { "Sid": "EcrPublicTokenAccess", "Effect": "Allow",
      "Action": ["ecr-public:GetAuthorizationToken"], "Resource": "*" },
    { "Sid": "StsForEcrPublicPull", "Effect": "Allow",
      "Action": ["sts:GetServiceBearerToken"], "Resource": "*" },
    { "Sid": "XRayTracingAccess", "Effect": "Allow",
      "Action": ["xray:PutTraceSegments","xray:PutTelemetryRecords",
                 "xray:GetSamplingRules","xray:GetSamplingTargets"], "Resource": "*" },
    { "Sid": "CloudWatchLogsGroup", "Effect": "Allow",
      "Action": ["logs:CreateLogGroup","logs:DescribeLogStreams"],
      "Resource": "arn:aws:logs:${AWS_REGION}:${ACCOUNT_ID}:log-group:/aws/bedrock-agentcore/runtimes/*" },
    { "Sid": "CloudWatchLogsDescribeGroups", "Effect": "Allow",
      "Action": ["logs:DescribeLogGroups"],
      "Resource": "arn:aws:logs:${AWS_REGION}:${ACCOUNT_ID}:log-group:*" },
    { "Sid": "CloudWatchLogsStream", "Effect": "Allow",
      "Action": ["logs:CreateLogStream","logs:PutLogEvents"],
      "Resource": "arn:aws:logs:${AWS_REGION}:${ACCOUNT_ID}:log-group:/aws/bedrock-agentcore/runtimes/*:log-stream:*" },
    { "Sid": "CloudWatchMetricsPublish", "Effect": "Allow", "Resource": "*",
      "Action": "cloudwatch:PutMetricData",
      "Condition": {"StringEquals": {"cloudwatch:namespace": "bedrock-agentcore"}} },
    { "Sid": "AgentCoreWorkloadIdentity", "Effect": "Allow",
      "Action": ["bedrock-agentcore:GetWorkloadAccessToken",
                 "bedrock-agentcore:GetWorkloadAccessTokenForJWT"],
      "Resource": ["arn:aws:bedrock-agentcore:${AWS_REGION}:${ACCOUNT_ID}:workload-identity-directory/default",
                   "arn:aws:bedrock-agentcore:${AWS_REGION}:${ACCOUNT_ID}:workload-identity-directory/default/workload-identity/harness_*"] },
    { "Sid": "AgentCoreBrowserDefault", "Effect": "Allow",
      "Action": ["bedrock-agentcore:StartBrowserSession","bedrock-agentcore:StopBrowserSession",
                 "bedrock-agentcore:GetBrowserSession","bedrock-agentcore:ListBrowserSessions",
                 "bedrock-agentcore:UpdateBrowserStream","bedrock-agentcore:ConnectBrowserAutomationStream",
                 "bedrock-agentcore:ConnectBrowserLiveViewStream"],
      "Resource": "arn:aws:bedrock-agentcore:${AWS_REGION}:aws:browser/*" },
    { "Sid": "AgentCoreCodeInterpreterDefault", "Effect": "Allow",
      "Action": ["bedrock-agentcore:StartCodeInterpreterSession","bedrock-agentcore:StopCodeInterpreterSession",
                 "bedrock-agentcore:GetCodeInterpreterSession","bedrock-agentcore:ListCodeInterpreterSessions",
                 "bedrock-agentcore:InvokeCodeInterpreter"],
      "Resource": "arn:aws:bedrock-agentcore:${AWS_REGION}:aws:code-interpreter/*" }
  ]
}
JSON

aws iam put-role-policy \
  --role-name "$ROLE_NAME" \
  --policy-name HarnessBaseline \
  --policy-document file://harness-policy.json
```

> **Note — Note the wildcard**
>
> `BedrockModelInvocation` as written allows every foundation model in every region. It is fine for a lab. In production, replace it with specific [inference profile](https://docs.aws.amazon.com/bedrock/latest/userguide/inference-profiles.html) ARNs paired with the foundation-model ARNs for each allowed region.

You will append optional statements to this role as the lab progresses — Memory in Lab 06, private ECR in Lab 07, Gateway and API keys in Labs 03–04. Keep `harness-policy.json` around.

### Step 4 — Enable Transaction Search (once per account)

Traces do not appear in the AgentCore Observability dashboard until CloudWatch Transaction Search is enabled. Do it now so Lab 08 has data.

```bash
aws xray update-trace-segment-destination --destination CloudWatchLogs
aws xray update-indexing-rule \
  --name "Default" \
  --rule '{"Probabilistic": {"DesiredSamplingPercentage": 100}}'
```

### Step 5 — Enable model access (once per account, per model)

IAM permission to call a model is not the same thing as *access* to it. Anthropic models on Bedrock are sold through AWS Marketplace, and your account must hold an **agreement** for each model before any principal — including an account administrator — can invoke it. A brand-new account holds none. Check first:

```bash
export MODEL_ID=anthropic.claude-sonnet-4-6

aws bedrock get-foundation-model-availability \
  --region "$AWS_REGION" \
  --model-id "$MODEL_ID"
```

Read `agreementAvailability.status`. `AVAILABLE` means you are done — skip to the checkpoint. `NOT_AVAILABLE` means no agreement exists yet, and every invocation will fail. Note that the other three fields can all look healthy while the agreement is missing:

*an account that cannot invoke the model*

```json
{
  "modelId": "anthropic.claude-sonnet-4-6",
  "agreementAvailability": { "status": "NOT_AVAILABLE" },
  "authorizationStatus": "AUTHORIZED",
  "entitlementAvailability": "AVAILABLE",
  "regionAvailability": "AVAILABLE"
}
```

To create the agreement, fetch the current Marketplace offer token for the model and accept it. The offer token is generated per request and expires, so always pipe a fresh one into the second call rather than pasting a saved value.

*accept the Marketplace offer*

```bash
OFFER_TOKEN="$(aws bedrock list-foundation-model-agreement-offers \
  --region "$AWS_REGION" \
  --model-id "$MODEL_ID" \
  --query 'offers[0].offerToken' --output text)"

aws bedrock create-foundation-model-agreement \
  --region "$AWS_REGION" \
  --model-id "$MODEL_ID" \
  --offer-token "$OFFER_TOKEN"
```

Inspect the pricing before you accept if you like — `list-foundation-model-agreement-offers` returns the full rate card under `offers[0].termDetails.usageBasedPricingTerm`. Accepting is usage-based with no upfront charge or commitment, and the agreement is reversible with `aws bedrock delete-foundation-model-agreement`.

The status now moves `NOT_AVAILABLE` → `PENDING` → `AVAILABLE`. Poll until it settles:

```bash
while true; do
  STATUS="$(aws bedrock get-foundation-model-availability \
    --region "$AWS_REGION" --model-id "$MODEL_ID" \
    --query 'agreementAvailability.status' --output text)"
  echo "$STATUS"
  [ "$STATUS" = "AVAILABLE" ] && break
  sleep 15
done
```

> **Warning — Two delays, not one**
>
> Reaching `AVAILABLE` takes roughly a minute, and invocations keep failing for *another* minute or two after that while the entitlement propagates to the runtime. If your first call still fails, wait five minutes and retry before changing anything — the most common way to lose an hour here is to start "fixing" IAM during that window.

Confirm with a real invocation. Note the model is invoked through an **inference profile** — the bare model id is rejected with `Invocation of model ID … with on-demand throughput isn't supported`, because these models are only reachable via a profile that fans out across regions. `global.` routes worldwide; `us.` stays inside the US regions.

*smoke test*

```bash
aws bedrock-runtime converse \
  --region "$AWS_REGION" \
  --model-id "global.${MODEL_ID}" \
  --messages '[{"role":"user","content":[{"text":"say hi"}]}]' \
  --inference-config '{"maxTokens":20}'
```

> **Note — Why the error message misleads**
>
> A missing agreement surfaces as `AccessDeniedException: Model access is denied due to IAM user or service role is not authorized to perform the required AWS Marketplace actions (aws-marketplace:ViewSubscriptions, aws-marketplace:Subscribe)`. That reads like an IAM problem, and the usual reflex is to start granting marketplace permissions to the execution role. It is almost always the wrong fix: Bedrock tries to auto-subscribe on your behalf, and reports the failure of that fallback rather than the missing agreement itself. You will see the identical error from a principal holding `AdministratorAccess`.
>
> Confirm which it is before touching any policy — `aws iam simulate-principal-policy --policy-source-arn <arn> --action-names aws-marketplace:Subscribe bedrock:InvokeModel` tells you whether IAM is genuinely denying you. If it returns `allowed`, the problem is the agreement.
>
> Once the agreement exists, no principal needs marketplace permissions at all — the subscribe path is never taken. The execution role you built in Step 3 has none, and that is correct.

The agreement is account-level and applies across regions, so you create it once even if you later move the harness between US regions or switch the profile prefix from `global.` to `us.`.

> **Checkpoint 01**
>
> `aws iam get-role --role-name $ROLE_NAME` returns a role whose trust policy names `bedrock-agentcore.amazonaws.com`, and `$ACCOUNT_ID` / `$ROLE_ARN` are set in your shell. `aws bedrock get-foundation-model-availability --model-id $MODEL_ID` reports `agreementAvailability.status: AVAILABLE`, and the `converse` smoke test returns text from the model.

## 02. Create and invoke your first harness

*~25 min · CreateHarness · InvokeHarness · streaming*

> **Goal**
>
> Stand up a harness with nothing but a name and a role, invoke it, and read the event stream. You will see that a harness with zero configuration is already a working agent — model, memory, shell, filesystem and observability are defaults, not decisions you have to make.

### Step 1 — Create

#### AgentCore CLI

```bash
# Scaffold a project. Creates the IAM role, harness and memory for you.
agentcore create --name researchagent --model-provider bedrock

# agentcore creates a directory for researchagent. 
# Change directory
cd researchagent

# Deploy the CloudFormation stack
agentcore deploy

# Confirm
agentcore status
```

Run `agentcore create` with no flags instead to get the interactive wizard, which walks you through project name, project type (choose **Harness**), model provider, environment, memory, and advanced settings. The wizard writes the same `app/researchagent/harness.json` that the flags produce.

#### AWS CLI

```bash
aws bedrock-agentcore-control create-harness \
  --harness-name "ResearchAgent" \
  --execution-role-arn "$ROLE_ARN"

# Capture the generated id — it has a random suffix
export HARNESS_ID="$(aws bedrock-agentcore-control list-harnesses \
  --query "harnesses[?harnessName=='ResearchAgent'].harnessId | [0]" --output text)"

# Poll until READY
aws bedrock-agentcore-control get-harness --harness-id "$HARNESS_ID" \
  --query '{status:status, arn:harnessArn}'

export HARNESS_ARN="$(aws bedrock-agentcore-control get-harness \
  --harness-id "$HARNESS_ID" --query harnessArn --output text)"
echo "$HARNESS_ARN"
```

### Step 2 — Invoke

A harness invocation needs a `runtimeSessionId`. Reuse it and the conversation continues in the same microVM, with the same filesystem and the same memory.

> **Warning — Session ID must be at least 33 characters**
>
> A UUID is 36 and works. A short string like `test-1` is rejected.

#### AgentCore CLI

```bash
export SESSION_ID="$(uuidgen)"

agentcore invoke --harness researchagent \
  --session-id "$SESSION_ID" \
  "Research three tropical vacation options under \$3k, within five hours of NYC."

# Same session — the agent still remembers the three options
agentcore invoke --harness researchagent \
  --session-id "$SESSION_ID" \
  "Which of those has the shortest flight?"
```

Add `--verbose` to print the raw streaming JSON events — useful when you want to see tool calls and reasoning blocks rather than just the final text.

#### boto3

*invoke.py*

```python
import os, sys, uuid, boto3

REGION      = os.environ.get("AWS_REGION", "us-west-2")
HARNESS_ARN = os.environ["HARNESS_ARN"]
SESSION_ID  = os.environ.get("SESSION_ID") or str(uuid.uuid4())

client = boto3.client("bedrock-agentcore", region_name=REGION)

def ask(prompt, **overrides):
    response = client.invoke_harness(
        harnessArn=HARNESS_ARN,
        runtimeSessionId=SESSION_ID,
        messages=[{"role": "user", "content": [{"text": prompt}]}],
        **overrides,
    )
    for event in response["stream"]:
        if "contentBlockDelta" in event:
            delta = event["contentBlockDelta"].get("delta", {})
            if "text" in delta:
                print(delta["text"], end="", flush=True)
        elif "contentBlockStart" in event:
            start = event["contentBlockStart"].get("start", {})
            if "toolUse" in start:
                print(f"\n  [tool: {start['toolUse']['name']}]", flush=True)
        elif "messageStop" in event:
            print(f"\n--- stopReason: {event['messageStop'].get('stopReason')}")
        elif "metadata" in event:
            print(f"--- usage: {event['metadata'].get('usage')}")
        elif "runtimeClientError" in event:
            print(f"\nError: {event['runtimeClientError']['message']}", file=sys.stderr)

if __name__ == "__main__":
    ask(" ".join(sys.argv[1:]) or "Research three tropical vacation options under $3k.")
```

```bash
export SESSION_ID="$(uuidgen)"
python invoke.py "Research three tropical vacation options under \$3k."
python invoke.py "Which of those has the shortest flight?"   # same session
```

### Step 3 — Read the stream

`InvokeHarness` returns events, not a blob. Learn these now; you will match against them in Labs 04 and 08.

| Event | Carries |
|---|---|
| `messageStart` | Beginning of a message, including `role`. |
| `contentBlockStart` | Start of a text, `toolUse` or `toolResult` block. Tool name and `toolUseId` arrive here. |
| `contentBlockDelta` | Incremental `text`, `toolUse` input fragments, or `reasoningContent`. |
| `contentBlockStop` | End of a content block. |
| `messageStop` | End of the message, including `stopReason`. |
| `metadata` | Token usage and latency metrics. |
| `runtimeClientError` | An execution error. |

The `stopReason` tells you why the loop ended — and three of the seven values are limits you will set yourself in Lab 08.

| stopReason | Meaning |
|---|---|
| `end_turn` | Finished normally. |
| `tool_use` | Waiting on a client-side inline function result (Lab 04). |
| `max_tokens` | The model's per-turn token limit was reached. |
| `max_iterations_exceeded` | Hit `maxIterations`. |
| `timeout_exceeded` | Hit `timeoutSeconds`. |
| `max_output_tokens_exceeded` | Exhausted the `maxTokens` budget. |
| `guardrail_intervened` | A Bedrock Guardrail blocked the exchange (Lab 03). |

### Step 4 — Optional: local dev loop

If you are on the AgentCore CLI, `agentcore dev` deploys your resources to AWS, then starts a local server and opens a browser inspector where you can chat with the harness, watch traces, and override configuration per session without redeploying.

```bash
agentcore dev                 # browser inspector
agentcore dev --no-browser    # terminal TUI instead
agentcore dev --logs          # non-interactive, logs to stdout
```

> **Checkpoint 02**
>
> The agent answered both prompts, and the second answer referred back to the first without you resending the history. That continuity is AgentCore Memory working by default — Lab 06 explains it.

## 03. Models and instructions

*~30 min · defaults vs overrides · provider switching · guardrails*

> **Goal**
>
> Internalise the central pattern of the harness: *defaults at creation time, overrides at invocation time*. You will set a system prompt and model as harness defaults, override both for a single call without changing the resource, then switch model providers mid-conversation and watch the context survive.

### Step 1 — Set defaults

If you never specify a model, the harness uses Claude Sonnet 4.6 on Bedrock (`global.anthropic.claude-sonnet-4-6`). Make that explicit and add instructions.

> **Warning — Do not guess the model id — list them**
>
> Anthropic model ids are inconsistent about the dated `-YYYYMMDD-v1:0` suffix. Sonnet 4.6 has none (`us.anthropic.claude-sonnet-4-6`); Sonnet 4.5 does (`us.anthropic.claude-sonnet-4-5-20250929-v1:0`); Opus 4.5 does (`us.anthropic.claude-opus-4-5-20251101-v1:0`). Inventing a plausible-looking suffix produces a runtime failure, not a deploy-time one — the harness deploys clean and fails on first invocation:
>
> `ValidationException: The provided model identifier is invalid.`
>
> Always copy the id from the live list rather than from memory or a blog post:

```bash
aws bedrock list-inference-profiles --region "$AWS_REGION" \
  --query "inferenceProfileSummaries[?contains(inferenceProfileId,'sonnet')].inferenceProfileId" \
  --output text
```

> **Warning — Every model needs its own agreement**
>
> Marketplace agreements are per model, not per account. The moment you point a harness at a model you have not used before — here, or in the per-call override in Step 2 — repeat Lab 01 Step 5 for that model id first, or the invocation fails with the misleading `aws-marketplace:ViewSubscriptions` error. Switching between inference profiles of the *same* model (`global.` ↔ `us.`) needs nothing new.

> **Warning — Names reject hyphens — and the two name rules differ**
>
> A hyphen anywhere in a harness or project name fails validation before anything is written:
>
> `ConfigValidationError: name: Must begin with a letter and contain only alphanumeric characters and underscores (max 40 chars)`
>
> The two rules are not the same, which is the part that catches people out:

| Name | Set by | Pattern | Underscores |
|---|---|---|---|
| Harness | `add harness --name`, `harness.json` | `^[a-zA-Z][a-zA-Z0-9_]{0,39}$` | Allowed — `research_agent` |
| Project | `agentcore create --name` | Letter, then alphanumerics only | **Rejected** — `researchagent` |

> **Warning — Why this bites**
>
> A hyphen is the natural thing to type, because CLI flags and most AWS resource names use them — and the CLI's own enum values are genuinely inconsistent about it. Credential types are hyphenated (`--type api-key`); tool types are not (`--type agentcore_browser`, `--tools agentcore_browser,agentcore_code_interpreter`). Read the enum in `--help` rather than inferring it from a neighbouring flag. Remember also that `agentcore create --name X` creates a project *and* a harness called `X`, so a project name has to satisfy the stricter of the two. Nothing is created when validation fails; correct the name and rerun.
>
> This lab uses `research_agent` for the harness and `researchagent` for the project throughout.

#### AgentCore CLI

```bash
agentcore add harness \
  --name research_agent \
  --model-id us.anthropic.claude-sonnet-4-6 \
  --system-prompt "You are a research assistant. Cite sources." \
  --tools agentcore_browser

agentcore deploy
```

To change defaults permanently later, edit `app/research_agent/harness.json` and run `agentcore deploy` again.

#### AWS CLI

```bash
aws bedrock-agentcore-control create-harness \
  --harness-name "research_agent" \
  --execution-role-arn "$ROLE_ARN" \
  --system-prompt '[{"text": "You are a research assistant. Cite sources."}]' \
  --tools '[{"type": "agentcore_browser", "name": "browser"}]'
```

To change defaults permanently later, call `update-harness` — which mints a new immutable version (Lab 09).

### Step 2 — Override for one call

The same fields you set as defaults can be passed per invocation. The harness resource is untouched; only this call sees the overrides. This is how you A/B a model or prompt in seconds instead of a redeploy.

#### boto3

```python
response = client.invoke_harness(
    harnessArn=HARNESS_ARN,
    runtimeSessionId=SESSION_ID,
    # These apply only to this call; the harness defaults stay intact
    model={"bedrockModelConfig": {"modelId": "us.anthropic.claude-opus-4-5-20251101-v1:0"}},
    systemPrompt=[{"text": "You are a terse research assistant. One paragraph answers only."}],
    tools=[
        {"type": "agentcore_browser", "name": "browser"},
        {"type": "agentcore_code_interpreter", "name": "code_interpreter"},
    ],
    messages=[{"role": "user", "content": [{"text": "Summarize this paper as a bullet list."}]}],
)
```

#### AgentCore CLI

```bash
# Switch the model for one call
agentcore invoke --harness research_agent \
  --model-id us.anthropic.claude-opus-4-5-20251101-v1:0 \
  "Summarize this research paper"

# Swap tools for one call
agentcore invoke --harness research_agent \
  --tools agentcore_browser,agentcore_code_interpreter \
  "Plot the citation counts as a bar chart"
```

### What can be overridden at invoke time

On the CLI: `--model-id`, `--tools`, `--system-prompt`, `--max-iterations`, `--max-tokens`, `--harness-timeout`, `--skills`, `--allowed-tools`, `--actor-id`. Add `--verbose` to see raw stream events.

### Step 3 — Choose the API format

Whenever your agent calls a model, two things have to be settled: the **protocol** (what shape the request and response JSON take) and the **endpoint** (which AWS service the traffic actually reaches).

You would expect a field called `apiFormat` to control only the first. On `bedrockModelConfig` it controls both — and that is the part worth remembering, because changing it moves your request to a different service.

| apiFormat | Protocol | Endpoint it routes to |
|---|---|---|
| `converse_stream` *(default)* | Bedrock Converse API | `bedrock-runtime` |
| `responses` | OpenAI-compatible Responses API | `bedrock-mantle` |
| `chat_completions` | OpenAI-compatible Chat Completions | `bedrock-mantle` |

`bedrock-mantle` is the OpenAI-compatible surface in Bedrock, and it supports a *different set of models and capabilities* than `bedrock-runtime`. So switching this field is not cosmetic. Three consequences you would otherwise meet as puzzling errors:

- **A model on one endpoint may not exist on the other.** Set `apiFormat: "responses"` with a model served only by `bedrock-runtime` and the call fails — you moved it to Mantle, where that model isn't offered.
- **Guardrails require `converse_stream`.** Not an arbitrary rule: `guardrailConfig` is a Converse API feature served by `bedrock-runtime`, so any other format routes you away from the endpoint that implements it. See Step 7.
- **`additionalParams` are protocol-shaped.** The Step 5 example passes `{"reasoning": {"effort": "high"}}`, an OpenAI Responses-shaped parameter — which is exactly why it is paired with `apiFormat: "responses"`.

On `openAiModelConfig` the field behaves ordinarily: it picks the protocol only — `responses` (default) or `chat_completions` — while the endpoint is a separate field of its own. That asymmetry is the whole thing to carry away: Bedrock bundles protocol and endpoint into one field, OpenAI splits them into two.

*OpenAI: the two choices are independent*

```python
"openAiModelConfig": {
    "modelId": "gpt-5.4",
    "apiFormat": "responses",          # protocol only
    "endpoint": {"bedrockMantle": {}}  # endpoint chosen separately
}
```

`apiFormat` does not apply to `gemini` or `lite_llm` at all.

```bash
# Bedrock model through the OpenAI-compatible Responses API
agentcore add harness --name responses_agent \
  --model-provider bedrock \
  --model-id us.anthropic.claude-sonnet-4-5-20250929-v1:0 \
  --api-format responses
agentcore deploy
```

A *new* harness, because `add harness` refuses a name that already exists. To put `research_agent` itself on the Responses format instead, set `"apiFormat": "responses"` inside `model` in `app/research_agent/harness.json` and redeploy.

### Step 4 — Store a third-party API key

Keys live in the AgentCore Identity token vault as an API key credential provider. The harness pulls the key at invocation time; your code never handles the raw secret.

#### AWS CLI

```bash
aws bedrock-agentcore-control create-api-key-credential-provider \
  --name my-openai-key \
  --api-key "$OPENAI_API_KEY"

export KEY_ARN="arn:aws:bedrock-agentcore:${AWS_REGION}:${ACCOUNT_ID}:token-vault/default/apikeycredentialprovider/my-openai-key"
```

#### AgentCore CLI

```bash
agentcore add credential --type api-key --name my-openai-key --api-key $OPENAI_API_KEY
agentcore deploy
```

Then grant the execution role permission to read it. Append to `harness-policy.json` and re-run `put-role-policy`:

*append to the policy Statement array*

```json
{
  "Sid": "AgentCoreApiKeyTokenVaultDefault",
  "Effect": "Allow",
  "Action": "bedrock-agentcore:GetResourceApiKey",
  "Resource": [
    "arn:aws:bedrock-agentcore:<region>:<accountId>:token-vault/default",
    "arn:aws:bedrock-agentcore:<region>:<accountId>:workload-identity-directory/default",
    "arn:aws:bedrock-agentcore:<region>:<accountId>:workload-identity-directory/default/workload-identity/harness_<agentName>-*"
  ]
},
{
  "Sid": "AgentCoreApiKeyTokenVaultPerKey",
  "Effect": "Allow",
  "Action": "bedrock-agentcore:GetResourceApiKey",
  "Resource": "arn:aws:bedrock-agentcore:<region>:<accountId>:token-vault/default/apikeycredentialprovider/<apiKeyName>"
},
{
  "Sid": "AgentCoreApiKeySecret",
  "Effect": "Allow",
  "Action": "secretsmanager:GetSecretValue",
  "Resource": "arn:aws:secretsmanager:<region>:<accountId>:secret:bedrock-agentcore-identity!default/apikey/<apiKeyName>-*"
}
```

> **Tip — Why the trailing dash-star**
>
> Secrets Manager appends a random suffix to secret ARNs, so the resource pattern must end in `-*`.

### Step 5 — Switch providers mid-session

This is the exercise that makes the harness click. Three turns, three different model configurations, *one* session ID. Context carries across all three.

#### boto3

```python
# Turn 1: Bedrock (native Converse API)
response = client.invoke_harness(
    harnessArn=HARNESS_ARN,
    runtimeSessionId=SESSION_ID,
    model={"bedrockModelConfig": {"modelId": "us.anthropic.claude-sonnet-4-5-20250929-v1:0"}},
    messages=[{"role": "user", "content": [{"text": "Analyze this codebase."}]}],
)

# Turn 2: Bedrock Mantle (OpenAI Responses format, no API key needed)
response = client.invoke_harness(
    harnessArn=HARNESS_ARN,
    runtimeSessionId=SESSION_ID,
    model={
        "bedrockModelConfig": {
            "modelId": "openai.gpt-oss-120b-1:0",
            "apiFormat": "responses",
            "additionalParams": {"reasoning": {"effort": "high"}},
        }
    },
    messages=[{"role": "user", "content": [{"text": "Now suggest fixes for the top three issues."}]}],
)

# Turn 3: OpenAI model via Bedrock Mantle (still no API key)
response = client.invoke_harness(
    harnessArn=HARNESS_ARN,
    runtimeSessionId=SESSION_ID,
    model={
        "openAiModelConfig": {
            "modelId": "gpt-5.4",
            "apiFormat": "responses",
            "endpoint": {"bedrockMantle": {}},
        }
    },
    messages=[{"role": "user", "content": [{"text": "Summarize the fixes as a bullet list."}]}],
)
```

`openAiModelConfig` with `"endpoint": {"bedrockMantle": {}}` reaches OpenAI models through Bedrock using your execution role — no API key. Use `apiKeyArn` instead to call OpenAI's own endpoint directly.

#### AgentCore CLI

```bash
SESSION_ID="$(uuidgen)"

# Turn 1: Bedrock, harness default API format
agentcore invoke --harness research_agent \
  --model-id us.anthropic.claude-sonnet-4-5-20250929-v1:0 \
  --session-id "$SESSION_ID" \
  "Analyze this codebase and identify performance bottlenecks."

# Turn 2: OpenAI direct, same session
agentcore invoke --harness research_agent \
  --model-provider open_ai \
  --model-id gpt-5.4 \
  --api-key-arn "$KEY_ARN" \
  --session-id "$SESSION_ID" \
  "Now suggest fixes for the top three issues."
```

> **Note — The CLI exposes fewer per-call overrides than the API**
>
> `agentcore invoke` overrides the model with `--model-id`, `--model-provider`, `--api-key-arn`, `--api-base` and `--additional-params` — but there is no `--api-format`. The boto3 tab sets `apiFormat` per call because `InvokeHarness` accepts it; the CLI does not surface it. To change the format, set it as a harness default with `agentcore add harness --api-format`, or call the API directly. Check `agentcore invoke --help` before assuming a boto3 field has a flag — the mapping is not one to one.

### Step 6 — Reach any provider through LiteLLM

`liteLlmModelConfig` covers everything LiteLLM supports, including OpenAI-compatible gateways. Set `modelId` to a provider-prefixed ID such as `gemini/gemini-2.5-pro` or `anthropic/claude-sonnet-4-6`.

| Field | Purpose |
|---|---|
| `modelId` *(required)* | LiteLLM provider-prefixed model ID. |
| `apiKeyArn` | Token-vault credential provider ARN. Required for key-authenticated providers; *not* needed for the `bedrock/` prefix, which uses the execution role. |
| `apiBase` | Custom endpoint URL for a proxy or self-hosted OpenAI-compatible gateway. |
| `additionalParams` | Passed through to LiteLLM unchanged. Read the warning below. |
| `maxTokens`, `temperature`, `topP` | Optional sampling controls. |

```bash
agentcore add harness --name proxy_agent \
  --model-provider lite_llm \
  --model-id openai/gpt-5.4 \
  --api-base https://my-llm-gateway.example.com/v1 \
  --api-key-arn "$KEY_ARN" \
  --additional-params '{"timeout": 30}'
agentcore deploy
```

> **Warning — `additionalParams` is a trust boundary**
>
> These parameters reach the provider unvalidated. A caller who controls them can redirect requests to another endpoint (`aws_bedrock_runtime_endpoint`), override the `Authorization` header (`extra_headers`), attempt an IAM role switch (`aws_role_name`), or change the target model and region entirely. If your application forwards caller-supplied model config to `InvokeHarness`, strip or allowlist it first. Lab 10 covers the mitigations.

### Step 7 — Apply a Bedrock Guardrail

Guardrails attach through `bedrockModelConfig.additionalParams` and require the `converse_stream` format — the default. Create a guardrail in the console or CLI first, then:

```python
import boto3

client = boto3.client("bedrock-agentcore")
response = client.invoke_harness(
    harnessArn=HARNESS_ARN,
    runtimeSessionId=SESSION_ID,
    model={
        "bedrockModelConfig": {
            "modelId": "us.amazon.nova-micro-v1:0",
            "apiFormat": "converse_stream",
            "additionalParams": {
                "guardrailConfig": {
                    "guardrailIdentifier": "arn:aws:bedrock:us-west-2:111122223333:guardrail/abc123def456",
                    "guardrailVersion": "1",
                    "trace": "enabled_full",
                }
            },
        }
    },
    messages=[{"role": "user", "content": [{"text": "Help me plan a trip."}]}],
)
```

*execution role addition*

```json
{
  "Effect": "Allow",
  "Action": "bedrock:ApplyGuardrail",
  "Resource": "arn:aws:bedrock:us-west-2:111122223333:guardrail/abc123def456"
}
```

Use a guardrail in the same region as the model request. When it intervenes, the stream reports `guardrail_intervened` as the stop reason — try a denied topic and confirm you see it.

> **Checkpoint 03**
>
> You ran two consecutive turns on one session ID with different `model` configurations, and the second turn referred correctly to the first turn's content. You can also state the difference between a default and an override without looking it up.

## 04. Tools

*~40 min · MCP · Gateway · Browser · Code Interpreter · inline functions*

> **Goal**
>
> Attach all five tool types, then restrict them. Tools in the harness are declarative — you list what the agent may call and AgentCore handles invocation, credentials and results. Four of the five run on the harness microVM; the fifth runs in your own process.

### The five types, plus two built-ins

| Type | What it is | Runs on |
|---|---|---|
| `remote_mcp` | Any remote Model Context Protocol endpoint, by URL. No Gateway needed. | Harness VM |
| `agentcore_gateway` | Governed connectivity with inbound/outbound auth, access control and Cedar policy. Every tool on the gateway becomes available. | Harness VM |
| `agentcore_browser` | Managed web browsing and automation. | Harness VM |
| `agentcore_code_interpreter` | Sandboxed Python / JavaScript / TypeScript execution. | Harness VM |
| `inline_function` | A tool *schema* only. The harness pauses and hands the call back to your code. | Your client |

Two built-ins are present in every session unless you restrict them: `shell` executes bash commands, and `file_operations` views, creates and edits files. They are why a bare harness can already write and run code.

### Step 1 — Attach the sandboxed tools

#### AgentCore CLI

```bash
agentcore add tool --harness research_agent \
  --type agentcore_browser --name browser

agentcore add tool --harness research_agent \
  --type agentcore_code_interpreter --name code-interpreter

agentcore deploy

# Exercise both
agentcore invoke --harness research_agent \
  "Look up the 5 largest US national parks by area, then compute the mean and plot a bar chart."
```

The `--type` flag uses underscore-separated names that match the type identifiers in `harness.json`.

#### boto3

```python
tools = [
    {"type": "agentcore_browser", "name": "browser"},
    {"type": "agentcore_code_interpreter", "name": "code_interpreter"},
]

response = client.invoke_harness(
    harnessArn=HARNESS_ARN,
    runtimeSessionId=SESSION_ID,
    tools=tools,
    messages=[{"role": "user", "content": [
        {"text": "Look up the 5 largest US national parks, then plot them as a bar chart."}
    ]}],
)
```

Watch the stream: you should see `contentBlockStart` events naming `browser` and `code_interpreter` before the final text. Both are already permitted by the baseline execution role from Lab 01.

### Step 2 — Connect a remote MCP server

Three authentication shapes, in increasing order of how much you should trust them in production.

```python
tools = [
    # a. Public MCP server, no auth
    {
        "type": "remote_mcp",
        "name": "exa",
        "config": {"remoteMcp": {"url": "https://mcp.exa.ai/mcp"}},
    },
    # b. Plain-text header — fine for a lab, not for production
    {
        "type": "remote_mcp",
        "name": "my-private-mcp",
        "config": {"remoteMcp": {
            "url": "https://mcp.example.com/api",
            "headers": {"Authorization": "Bearer <your-token>"}
        }},
    },
    # c. Key held in the AgentCore Identity token vault. The ${arn:...} reference
    #    is resolved to the real key at invocation time.
    {
        "type": "remote_mcp",
        "name": "exa-secure",
        "config": {"remoteMcp": {
            "url": "https://mcp.exa.ai/mcp",
            "headers": {"x-api-key": "${arn:aws:bedrock-agentcore:us-west-2:123456789012:token-vault/default/apikeycredentialprovider/my-exa-key}"}
        }},
    },
]
```

*the unauthenticated case, on the CLI*

```bash
agentcore add tool --harness research_agent --type remote_mcp \
  --name exa --url https://mcp.exa.ai/mcp

agentcore deploy
```

> **Warning — `add tool` cannot set MCP headers**
>
> There is no `--header` on `agentcore add tool` (0.24.x) — check `agentcore add tool --help`. The `-H, --header` you may remember belongs to `agentcore invoke`, where it sets headers on *your* request to the runtime, not on the agent's outbound MCP calls. Two supported routes for an authenticated server:

*headers at harness-creation time*

```bash
agentcore add harness --name secure_mcp_agent \
  --tools remote_mcp \
  --mcp-name exa-secure \
  --mcp-url https://mcp.exa.ai/mcp \
  --mcp-headers '{"x-api-key": "${arn:aws:bedrock-agentcore:us-west-2:123456789012:token-vault/default/apikeycredentialprovider/my-exa-key}"}'
```

Or, to add a secured server to a harness that already exists, edit its `harness.json` directly — the `tools` array takes the same shape as the boto3 block above — then `agentcore deploy`.

> **Note — When to reach for Gateway instead**
>
> For managed credential rotation and OAuth-protected tools, put the MCP server behind AgentCore Gateway and use AgentCore Identity rather than raw headers.

### Step 3 — Attach a Gateway

A gateway is a governed tool surface: reference one ARN and every tool configured on it becomes available to the agent, with Cedar policies gating each call.

```python
# SigV4 (AWS IAM) outbound auth — the default
{
    "type": "agentcore_gateway",
    "name": "my-gateway",
    "config": {"agentCoreGateway": {
        "gatewayArn": "arn:aws:bedrock-agentcore:us-west-2:123456789012:gateway/my-gateway"
    }},
},
# OAuth outbound auth
{
    "type": "agentcore_gateway",
    "name": "my-oauth-gateway",
    "config": {"agentCoreGateway": {
        "gatewayArn": "arn:aws:bedrock-agentcore:us-west-2:123456789012:gateway/my-oauth-gateway",
        "outboundAuth": {"oauth": {
            "credentialProviderName": "my-oauth-provider",
            "scopes": ["read", "write"]
        }}
    }},
}
```

```bash
# By ARN
agentcore add tool --harness research_agent --type agentcore_gateway \
  --name my-gateway \
  --gateway-arn arn:aws:bedrock-agentcore:us-west-2:123456789012:gateway/my-gateway

# Or by project-local name
agentcore add tool --harness research_agent --type agentcore_gateway \
  --name my-gateway --gateway my-gateway
```

The execution role needs `bedrock-agentcore:InvokeGateway` on the gateway ARN:

*execution role addition*

```json
{
  "Sid": "AgentCoreGatewayAccess",
  "Effect": "Allow",
  "Action": "bedrock-agentcore:InvokeGateway",
  "Resource": "arn:aws:bedrock-agentcore:<region>:<accountId>:gateway/<gatewayId>"
}
```

### Step 4 — Optional: give the agent web search

> **Warning — us-east-1 only**
>
> The Web Search Tool connector is available in US East (N. Virginia). Create both the gateway and the harness there, or skip this step.

Web search is not a harness tool type — it is a Gateway *connector*. The path is: create a gateway with MCP protocol and `AWS_IAM` inbound auth, add a target with `connectorId: "web-search"`, then attach that gateway to your harness. The gateway exposes a standard MCP `WebSearch` tool the agent discovers like any other. Queries are served entirely within AWS.

1. Create the Gateway and connector target. Its *Gateway service role* needs `bedrock-agentcore:InvokeWebSearch` on the connector. Wait for `READY` and note the ARN.
2. Grant the *harness execution role* — a different role — `bedrock-agentcore:InvokeGateway` on that gateway ARN.
3. Attach the gateway as an `agentcore_gateway` tool with default `AWS_IAM` outbound auth.

```python
tools = [
    {
        "type": "agentcore_gateway",
        "name": "web-search",
        "config": {"agentCoreGateway": {
            "gatewayArn": "arn:aws:bedrock-agentcore:us-east-1:123456789012:gateway/my-web-search-gateway",
            "outboundAuth": {"awsIam": {}}
        }},
    },
]

client.create_harness(
    harnessName="research_agent",
    executionRoleArn="arn:aws:iam::123456789012:role/MyHarnessRole",
    tools=tools,
)
```

```bash
agentcore add tool --harness research_agent --type agentcore_gateway \
  --name web-search \
  --gateway-arn arn:aws:bedrock-agentcore:us-east-1:123456789012:gateway/my-web-search-gateway

agentcore deploy
agentcore invoke --harness research_agent \
  "Search the web for the latest AWS announcements and cite your sources."
```

> **Tip — Also grant Memory**
>
> If your harness uses managed memory — the default — the execution role needs the AgentCore Memory permissions too, because the agent reads and writes session memory on every invocation. Lab 06 adds them.

### Step 5 — Inline functions and human-in-the-loop

An inline function is a schema with no implementation on the AWS side. When the agent calls it, the stream stops with `stopReason: "tool_use"` and hands you the call. You decide what happens — prompt a human, hit an internal API — then send the result back.

This is a three-part exercise. First, declare the tool and invoke:

*1. invoke with an inline tool*

```python
response = client.invoke_harness(
    harnessArn=HARNESS_ARN,
    runtimeSessionId=SESSION_ID,
    tools=[{
        "type": "inline_function",
        "name": "get_weather",
        "config": {"inlineFunction": {
            "description": "Get the current weather for a city.",
            "inputSchema": {
                "type": "object",
                "properties": {"city": {"type": "string"}},
                "required": ["city"]
            }
        }}
    }],
    messages=[{"role": "user", "content": [{"text": "What's the weather in Seattle?"}]}],
)
```

*2. capture the call from the stream*

```python
tool_use_id = None
tool_name = None
tool_input = None
for event in response["stream"]:
    if "contentBlockStart" in event:
        start = event["contentBlockStart"].get("start", {})
        if "toolUse" in start and start["toolUse"].get("name") == "get_weather":
            tool_use_id = start["toolUse"]["toolUseId"]
            tool_name = start["toolUse"]["name"]
    if "contentBlockDelta" in event:
        delta = event["contentBlockDelta"].get("delta", {})
        if "toolUse" in delta:
            tool_input = (tool_input or "") + delta["toolUse"].get("input", "")
```

*3. execute it yourself and send the result back*

```python
import json

client.invoke_harness(
    harnessArn=HARNESS_ARN,
    runtimeSessionId=SESSION_ID,
    messages=[
        {
            "role": "assistant",
            "content": [{"toolUse": {"toolUseId": tool_use_id,
                                     "name": tool_name,
                                     "input": json.loads(tool_input)}}],
        },
        {
            "role": "user",
            "content": [{
                "toolResult": {
                    "toolUseId": tool_use_id,
                    "content": [{"text": "72°F, partly cloudy"}],
                    "status": "success",
                }
            }],
        },
    ],
)
```

> **Note — You must send both messages**
>
> The assistant `toolUse` message *and* your `toolResult`. The harness deliberately does not persist the inline-function turn: if the client never returned a result, a stored assistant `toolUse` with no matching `toolResult` would leave the session corrupted. Requiring both keeps the session clean whether or not you complete the call.

On the CLI, declare the tool then fill in its schema in `harness.json`:

```bash
agentcore add tool --harness research_agent --type inline_function \
  --name approve_purchase \
  --description "Request human approval for a purchase" \
  --input-schema '{"type":"object","properties":{"item":{"type":"string"},"amount":{"type":"number"}},"required":["item","amount"]}'
agentcore deploy

agentcore invoke --harness research_agent \
  "Find a mechanical keyboard under \$200 and request approval."
```

In the TUI the invocation pauses and prompts you for the tool result inline. In non-interactive CLI mode the stream returns `stopReason: "tool_use"` and you reply with a follow-up invoke.

### Step 6 — Restrict what the agent may call

`allowedTools` scopes which tools the model can select. Omit it and everything is allowed — including `shell`.

| Pattern | Example | Matches |
|---|---|---|
| `*` | `"*"` | All tools |
| Plain name | `"shell"` | A built-in by name |
| Built-in glob | `"file_*"` | `file_operations`, `file_read` |
| `@builtin` | `"@builtin"` | All built-in tools |
| `@builtin/name` | `"@builtin/shell"` | One specific built-in |
| `@server` | `"@git"` | All tools from one MCP server |
| `@server/tool` | `"@git/git_status"` | One specific MCP tool |
| `@server/glob` | `"@git/read_*"` | A glob within a server |
| `@*/tool` | `"@*-mcp/status"` | A glob across servers |

*try to make the agent break its own restriction*

```bash
agentcore invoke --harness research_agent \
  --allowed-tools "@builtin/file_operations,agentcore_code_interpreter" \
  "Run 'whoami' in the shell, then tell me what it printed."
```

The agent should decline or route around it — `shell` is not in the allowlist.

> **Warning — `allowedTools` is not a security boundary for direct execution**
>
> It scopes *LLM tool selection* during `InvokeHarness` only. It has no effect on `InvokeAgentRuntimeCommand`, which is a separate API with its own IAM action (`bedrock-agentcore:InvokeAgentRuntimeCommand`) that runs commands directly on the microVM without passing through the model. To prevent direct command execution, do not grant that action in your IAM policies.

> **Checkpoint 04**
>
> You saw a `toolUse` block in the stream for a sandboxed tool, completed one full inline-function round trip (invoke → capture → return result), and confirmed that an `allowedTools` list actually stopped the agent from reaching `shell`.

## 05. Skills

*~25 min · AWS skills · Git · S3 · filesystem paths*

> **Goal**
>
> Give the agent domain expertise on demand from four different sources, and understand progressive disclosure — why attaching twenty skills does not cost you twenty skills' worth of context window.

A skill is a bundle of markdown and scripts following the open [AgentSkills.io](https://agentskills.io/specification) standard: a `SKILL.md` with YAML frontmatter (name, description) plus optional `scripts/`, `references/` and `assets/` directories.

> **Note — Progressive disclosure**
>
> Only the metadata — roughly 100 tokens per skill — is injected into the system prompt. Full instructions load on demand via a tool call when the agent decides the skill is relevant. This is what lets you attach a broad catalogue without flooding the context window.

Skills are fetched once per session on the first invocation and persist on disk for the rest of it. When the microVM expires and a new session begins, they are re-fetched to guarantee freshness. Skills set on the harness are defaults; invoke-time skills are *appended* after them, and on a name collision the invoke-time version wins.

| Source | What it does | Reach for it when |
|---|---|---|
| AWS Skills | Pre-built skills for AWS services from the AWS Agent Toolkit, selected by glob. | You want ready-made AWS expertise with zero setup. |
| Git (HTTPS) | Clone from any public or private Git repo, with sparse checkout for subdirectories. | Your skills live in GitHub or GitLab and you don't want an S3 copy. |
| Amazon S3 | Fetch from your own bucket using the execution role. | You need control over versioning, encryption and access governance. |
| Path | Reference a skill already on the harness filesystem. | It is baked into your container image or installed at session start. |

### Step 1 — Enable AWS skills

Organised hierarchically and selected with glob patterns.

| Category | Pattern | Typical contents |
|---|---|---|
| Core | `core-skills/*` | EC2, S3, Lambda, DynamoDB, CloudWatch, IAM |
| Analytics | `specialized-skills/analytics-skills/*` | Athena, Glue, QuickSight, data lakes |
| Operations | `specialized-skills/operations-skills/*` | Troubleshooting, diagnostics, log analysis |
| Storage | `specialized-skills/storage-skills/*` | S3, EFS, FSx, Backup |

*everything*

```bash
aws bedrock-agentcore-control create-harness \
  --harness-name "MyHarness" \
  --execution-role-arn "${ROLE_ARN}" \
  --skills '[{"awsSkills": {}}]'
```

*by category*

```bash
aws bedrock-agentcore-control create-harness \
  --harness-name "MyHarness" \
  --execution-role-arn "${ROLE_ARN}" \
  --skills '[{"awsSkills": {"paths": ["core-skills/*", "specialized-skills/operations-skills/*"]}}]'
```

*one skill, and a combined selection*

```python
response = client.invoke_harness(
    harnessArn=HARNESS_ARN,
    runtimeSessionId=SESSION_ID,
    skills=[{"awsSkills": {"paths": ["core-skills/aws-cdk"]}}],
    messages=[{"role": "user", "content": [{"text": "Create a CDK stack for a Lambda function."}]}],
)

# Combine patterns freely
skills=[{"awsSkills": {"paths": [
    "core-skills/aws-cdk",
    "core-skills/aws-serverless",
    "specialized-skills/storage-skills/*",
]}}]
```

Paths must be relative — no leading `/`, no `..`. A glob that matches nothing fails the invocation rather than silently doing less than you asked. Multiple `awsSkills` entries in one payload are merged.

### Step 2 — Pull a skill from Git

#### AgentCore CLI

```bash
# Public repo, single skill from a subdirectory
agentcore add skill --harness research_agent \
  --git https://github.com/anthropics/skills \
  --git-path skills/docx
agentcore deploy

# Private repo — --credential names an API key credential holding a PAT
agentcore add skill --harness research_agent \
  --git https://github.com/my-org/internal-skills \
  --git-path excel \
  --credential my-github-pat
agentcore deploy
```

Remove with `agentcore remove skill` using the same source flags. To override skills for one call without changing the harness, use `agentcore invoke --skills <sources>` — comma-separated paths, `s3://` URIs or `https://` Git URLs. Git authentication is *not* supported on the invoke override.

#### boto3

```python
# Public repository
skills=[{"git": {"url": "https://github.com/anthropics/skills", "path": "skills/docx"}}]

# Private repository — PAT stored in AgentCore Identity
skills=[
    {
        "git": {
            "url": "https://github.com/my-org/internal-skills",
            "path": "excel",
            "auth": {
                "credentialArn": "arn:aws:bedrock-agentcore:us-west-2:123456789012:token-vault/default/apikeycredentialprovider/my-github-pat"
            },
        }
    }
]
```

- `url` *(required)* — HTTPS URL of the repository.
- `path` — subdirectory containing the skill; repository root if omitted.
- `auth.credentialArn` — API key credential provider holding a personal access token.
- `auth.username` — git username, defaults to `oauth2`.

> **Warning — Git needs egress and finishes in 60 seconds**
>
> The clone must complete within 60s or the invocation fails. If your harness runs in a VPC, the subnet needs a NAT gateway — the same requirement as remote MCP servers and custom container pulls.

### Step 3 — Fetch a skill from S3

Upload a skill directory to a bucket you own, then reference it. S3 sources work through S3 VPC endpoints, so unlike Git they need no NAT gateway.

```bash
export SKILL_BUCKET="agentcore-harness-lab-${ACCOUNT_ID}"
aws s3 mb "s3://${SKILL_BUCKET}"

mkdir -p company-style
cat > company-style/SKILL.md <<'MD'
---
name: company-style
description: Use when drafting any customer-facing summary or report.
---

# House style

- Lead with the conclusion, then the evidence.
- No adjectives in headlines. No exclamation marks.
- Every number carries its unit and its as-of date.
- Prefer "we found" over "it was found".
MD

aws s3 cp company-style/ "s3://${SKILL_BUCKET}/skills/company-style/" --recursive
```

```python
response = client.invoke_harness(
    harnessArn=HARNESS_ARN,
    runtimeSessionId=SESSION_ID,
    skills=[{"s3": {"uri": "s3://my-skills-bucket/skills/company-style/"}}],
    messages=[{"role": "user", "content": [{"text": "Draft a summary following our style guide."}]}],
)
```

```bash
agentcore add skill --harness research_agent \
  --s3 s3://my-skills-bucket/skills/company-style/
agentcore deploy
```

The execution role needs read access. Each S3 skill must be 1 GB or smaller.

*execution role addition*

```json
{
  "Sid": "AgentCoreSkillS3Access",
  "Effect": "Allow",
  "Action": ["s3:GetObject", "s3:ListBucket"],
  "Resource": [
    "arn:aws:s3:::<skillBucket>",
    "arn:aws:s3:::<skillBucket>/*"
  ]
}
```

### Step 4 — Reference a skill already on the filesystem

Two ways to get it there. Bake it into a custom image (Lab 07):

```dockerfile
COPY skills/xlsx .agents/skills/xlsx
```

Or install it at session start, before the first agent invocation:

```bash
agentcore invoke --exec --harness research_agent --session-id "$SESSION" \
  "git clone --depth 1 https://github.com/anthropics/skills /tmp/skills && cp -r /tmp/skills/skills/xlsx .agents/skills/xlsx"
```

*then reference it*

```python
skills=[{"path": ".agents/skills/xlsx"}]
```

### Step 5 — Combine all four

They coexist in a single payload. This is the shape a mature agent ends up with: AWS expertise, an open-source capability, your house style, and something from your own image.

```python
response = client.invoke_harness(
    harnessArn=HARNESS_ARN,
    runtimeSessionId=SESSION_ID,
    skills=[
        {"awsSkills": {"paths": ["core-skills/aws-cdk"]}},
        {"git": {"url": "https://github.com/anthropics/skills", "path": "skills/docx"}},
        {"s3": {"uri": "s3://my-bucket/skills/company-style/"}},
        {"path": ".agents/skills/xlsx"},
    ],
    messages=[{"role": "user", "content": [{"text": "Help me with this project."}]}],
)
```

### Failure modes

Every fetch failure fails the invocation with a descriptive error. Skills are never silently skipped — which is what you want, because a silently-missing skill produces a plausible but unguided answer.

| Failure | Error |
|---|---|
| S3 access denied | `Failed to fetch skill: AccessDeniedException. Ensure execution role has s3:GetObject permission.` |
| S3 object not found | `Skill source not found: s3://…` |
| Git clone network failure | `Failed to clone skill: could not resolve host` |
| Git auth denied | `Failed to clone skill: authentication failed` |
| Git path missing in repo | `Skill path 'x' not found in repository` |
| Git timeout | `Failed to clone skill: operation timed out after 60s` |
| Over the size limit | `Skill exceeds 1GB size limit` |
| Glob matched nothing | `AWS skill path 'x' matched no skills` |
| Path traversal attempt | `Invalid AWS skill path: must be a relative path without '..'` |
| Bundle missing in runtime | `AWS Skills are not available in this runtime (missing directory: /opt/amazon/skills)` |

> **Warning — Skills are trusted input**
>
> The harness does not validate, sanitize or inspect skill content before handing it to the agent — and skills can be overridden per invocation. If your application forwards caller-supplied fields to `InvokeHarness`, a caller can point the harness at their own S3 prefix or Git repo containing arbitrary instructions and scripts. Strip the `skills` field from untrusted requests, or allowlist permitted prefixes and repositories.

> **Checkpoint 05**
>
> You attached at least two sources — one AWS skills glob and one of Git or S3 — and the agent's output visibly changed to follow the attached guidance. Bonus: you triggered one of the errors above on purpose and recognised it.

## 06. Memory

*~30 min · short/long term · strategies · actor scoping · truncation*

> **Goal**
>
> Understand why Lab 02's second question worked, then take control of it: tune strategies, isolate memory per user with `actorId`, attach your own Memory resource, and decide what happens when a conversation outgrows the context window.

On every invocation the harness persists the conversation to AgentCore Memory, scoped by session ID — and by actor ID when you supply one. On the next invocation with the same session ID, history is loaded *before* the agent reasons. You never resend previous messages; you send only the new one. This survives the microVM expiring.

### The three concepts

- **Short-term memory** — raw events (messages, tool calls) within a session. This is what gives continuity across turns.
- **Long-term memory** — durable knowledge extracted by configurable strategies and retrievable by semantic search in *later* sessions.
- **Actor ID** — who is interacting: a user, another agent, a system. Events are scoped by actorId + sessionId, so each actor's memory is isolated. Long-term retrieval uses actorId as a template variable in namespace paths such as `/summary/{actorId}/{sessionId}/`.

### Step 1 — Managed memory, and tuning it

By default the harness provisions a Memory instance for you with semantic + summarization strategies and 30-day event expiry. Nothing to create. To customise at create time:

```bash
aws bedrock-agentcore-control create-harness \
  --harness-name "MyHarness" \
  --execution-role-arn "$ROLE_ARN" \
  --memory '{"managedMemoryConfiguration": {"strategies": ["SEMANTIC", "SUMMARIZATION", "USER_PREFERENCE"], "eventExpiryDuration": 60}}'
```

*add a strategy later*

```bash
aws bedrock-agentcore-control update-harness \
  --harness-id "$HARNESS_ID" \
  --memory '{"optionalValue": {"managedMemoryConfiguration": {"strategies": ["SEMANTIC", "SUMMARIZATION", "USER_PREFERENCE", "EPISODIC"]}}}'
```

*AgentCore CLI*

```bash
agentcore create --name myagent          # memory enabled by default
agentcore create --name myagent --no-harness-memory   # skip it
agentcore deploy
```

| Strategy | What it extracts |
|---|---|
| `SEMANTIC` | Factual knowledge from conversations, retrievable by semantic search. |
| `SUMMARIZATION` | Running summaries, scoped by actor and session. |
| `USER_PREFERENCE` | Preferences and settings expressed during conversation. |
| `EPISODIC` | Significant events and experiences as discrete episodes. |

Grant the execution role access to memory operations:

*execution role addition*

```json
{
  "Sid": "AgentCoreMemory",
  "Effect": "Allow",
  "Action": [
    "bedrock-agentcore:CreateEvent",
    "bedrock-agentcore:DeleteEvent",
    "bedrock-agentcore:GetEvent",
    "bedrock-agentcore:ListEvents",
    "bedrock-agentcore:RetrieveMemoryRecords"
  ],
  "Resource": "arn:aws:bedrock-agentcore:<region>:<accountId>:memory/<memoryId>"
}
```

> **Note — Managed memory has guardrails on its lifecycle**
>
> Strategy configuration is controlled through `UpdateHarness`, though you can still read and write events and query records directly through the Memory APIs. It cannot be deleted directly through the Memory APIs. To turn it into an ordinary Memory resource, either switch the harness to BYO or disabled with `UpdateHarness`, or pass `deleteManagedMemory=false` on deletion to disassociate instead — `DeleteHarness` cascade-deletes it by default.

### Step 2 — Prove per-user isolation

Run these in order. Two actors, *one* session ID.

```python
def ask_as(actor, text):
    return client.invoke_harness(
        harnessArn=HARNESS_ARN,
        runtimeSessionId=SESSION_ID,
        actorId=actor,
        messages=[{"role": "user", "content": [{"text": text}]}],
    )

ask_as("user-123", "Remember that I always want answers in metric units.")
ask_as("user-456", "What units do I prefer?")     # should not know
ask_as("user-123", "What units do I prefer?")     # should say metric
```

### Step 3 — Bring your own Memory

Attach an existing Memory instance when you need custom namespace templates, KMS encryption, or memory shared across several harnesses.

```bash
aws bedrock-agentcore-control create-memory \
  --name "MyMemory" \
  --event-expiry-duration 30 \
  --description "Memory for my harness"

aws bedrock-agentcore-control update-harness \
  --harness-id "$HARNESS_ID" \
  --memory '{"optionalValue": {"agentCoreMemoryConfiguration": {"arn": "arn:aws:bedrock-agentcore:us-west-2:123456789012:memory/MyMemory-abc123"}}}'
```

*or on the CLI*

```bash
agentcore add harness --name myagent \
  --memory-mode existing \
  --memory-arn "arn:aws:bedrock-agentcore:us-west-2:123456789012:memory/MyMemory-abc123"
agentcore deploy
```

`--memory-arn` lives on `add harness`, not on `agentcore create` — `create` only takes `--memory none|shortTerm|longAndShortTerm`, which provisions a *new* managed memory rather than attaching an existing one. Pair the ARN with `--memory-mode existing`.

*or turn memory off entirely*

```bash
aws bedrock-agentcore-control update-harness \
  --harness-id "$HARNESS_ID" \
  --memory '{"optionalValue": {"disabled": {}}}'
```

### Step 4 — Long-term retrieval

When a harness has active strategies, retrieval is automatic: the harness derives a retrieval configuration from the Memory instance's active strategies and, on each invocation, queries relevant long-term memories and injects them into context before reasoning. Defaults are `topK=10` and `relevanceScore=0.2` per namespace, for managed and BYO alike.

Supply `retrievalConfig` explicitly and your values take priority — no automatic derivation happens at all. Use it to choose namespaces, tune the knobs, or disable retrieval for a specific strategy.

```bash
aws bedrock-agentcore-control update-harness \
  --harness-id "$HARNESS_ID" \
  --memory '{"optionalValue": {"agentCoreMemoryConfiguration": {
      "arn": "arn:aws:bedrock-agentcore:us-west-2:123456789012:memory/MyMemory-abc123",
      "retrievalConfig": {
        "/facts/{actorId}/": {"topK": 5, "relevanceScore": 0.5, "strategyId": "FactExtractor-abc123"}
      }}}}'
```

> **Warning — Refresh retrieval after changing BYO strategies**
>
> If you add or remove strategies on a BYO Memory instance *after* attaching it, call `UpdateHarness` to refresh the derived retrieval configuration. Managed memory refreshes automatically when strategies change through `UpdateHarness`.

### Step 5 — Decide what happens when context overflows

| Strategy | Behaviour |
|---|---|
| `sliding_window` *(default)* | Keeps the most recent N messages. Simple and predictable. |
| `summarization` | Compresses older messages into a summary — more context in fewer tokens. |
| `none` | No truncation. Only if you manage context size yourself. |

```bash
aws bedrock-agentcore-control update-harness \
  --harness-id "$HARNESS_ID" \
  --truncation '{"strategy": "sliding_window", "slidingWindowConfig": {"numMessages": 30}}'
```

> **Checkpoint 06**
>
> `user-456` did not know `user-123`'s preference, and `user-123` did. You can also say what changes when you set `retrievalConfig` explicitly rather than leaving it to derivation.

## 07. Environment and filesystem

*~35 min · direct shell · custom containers · EFS · S3 Files · VPC*

> **Goal**
>
> Work with the microVM directly. You will run commands on it without going through the model at all, package your own environment, and mount storage that outlives the session.

### Step 1 — Run commands without the agent loop

Not everything needs to go through the model. `InvokeAgentRuntimeCommand` gives you a shell on the harness microVM: deterministic execution, no reasoning, no token cost, no ambiguity. Use it to prepare the environment before an invocation, act on what the agent produced afterwards, or just inspect the VM while developing.

#### AgentCore CLI

```bash
export SESSION_ID="$(uuidgen)"

# Prepare: install dependencies before the agent starts
agentcore invoke --exec --harness research_agent --session-id "$SESSION_ID" \
  "pip install pandas matplotlib"

# Let the agent work
agentcore invoke --harness research_agent --session-id "$SESSION_ID" \
  "Load /tmp/input.csv, compute monthly totals, save to /tmp/results.csv."

# Act on the output — same session, so same filesystem
agentcore invoke --exec --harness research_agent --session-id "$SESSION_ID" \
  "ls -la /tmp && cat /tmp/results.csv"
```

In the TUI, press `!` to enter exec mode and run commands inline.

#### boto3

```python
response = client.invoke_agent_runtime_command(
    agentRuntimeArn=HARNESS_ARN,
    runtimeSessionId=SESSION_ID,
    body={"command": "ls -la /workspace"},
)

for event in response["stream"]:
    chunk = event.get("chunk", {})
    if "contentDelta" in chunk:
        delta = chunk["contentDelta"]
        if "stdout" in delta:
            print(delta["stdout"], end="", flush=True)
        if "stderr" in delta:
            print(delta["stderr"], end="", flush=True)
    elif "contentStop" in chunk:
        print(f"\n[exit code: {chunk['contentStop']['exitCode']}]")
```

The base environment ships Python and bash. For `git`, `node` or anything else, install it at session start or bake a custom image.

> **Warning — Commands run as root, and this API bypasses the model**
>
> Commands execute as uid 0 in the microVM — analogous to root on your own EC2 instance, where the IAM permission is the access gate rather than the in-VM privilege level. A `USER` directive in your Dockerfile applies to the agent process only; `InvokeAgentRuntimeCommand` runs at a higher privilege, like `docker exec` defaulting to root. It also does not respect `allowedTools`. If you do not want direct command execution, do not grant `bedrock-agentcore:InvokeAgentRuntimeCommand`.

### Step 2 — Set environment variables

Passed to the runtime container and visible to the agent and any custom container in the session.

```bash
aws bedrock-agentcore-control create-harness \
  --harness-name "MyHarness" \
  --execution-role-arn "$ROLE_ARN" \
  --environment-variables '{"MY_API_URL": "https://api.example.com", "LOG_LEVEL": "debug"}'
```

*or in harness.json*

```json
{
  "environmentVariables": {
    "MY_API_URL": "https://api.example.com",
    "LOG_LEVEL": "debug"
  }
}
```

Verify with the exec API — a good use of Step 1:

```bash
agentcore invoke --exec --harness research_agent --session-id "$SESSION_ID" "env | grep MY_API_URL"
```

### Step 3 — Bring your own container

When Python and bash are not enough, package your source, dependencies, runtimes and tools into an image, push it to ECR, and point the harness at it.

> **Warning — Two rules that catch people out**
>
> Images must be built for `linux/arm64`. And the harness *overrides your `ENTRYPOINT` and `CMD`* — it keeps the container running as an environment, so your startup command never executes. Your installed software, filesystem and environment variables are all available; if you need a background process such as a dev server, start it with `InvokeAgentRuntimeCommand` after the session begins.

```dockerfile
# syntax=docker/dockerfile:1
FROM --platform=linux/arm64 public.ecr.aws/docker/library/python:3.12-slim

RUN apt-get update && apt-get install -y --no-install-recommends \
      git curl jq ripgrep && rm -rf /var/lib/apt/lists/*

RUN pip install --no-cache-dir pandas matplotlib duckdb

# Bake in a skill so it can be referenced by path (Lab 05, Step 4)
COPY skills/xlsx .agents/skills/xlsx
```

#### AgentCore CLI

```bash
# The CLI builds, pushes to ECR and attaches on deploy
agentcore create --name codingagent --container ./Dockerfile
agentcore deploy

# Or reference a pre-built public image
agentcore create --name nodeagent \
  --container public.ecr.aws/docker/library/node:slim
agentcore deploy
```

#### AWS CLI

```bash
aws bedrock-agentcore-control create-harness \
  --harness-name "CodingAgent" \
  --execution-role-arn "$ROLE_ARN" \
  --environment-artifact '{"containerConfiguration": {"containerUri": "123456789012.dkr.ecr.us-west-2.amazonaws.com/my-dev-env:latest"}}' \
  --system-prompt '[{"text": "You are an expert TypeScript developer."}]'
```

A private ECR image needs pull permissions on the execution role:

*execution role addition*

```json
{
  "Sid": "ECRImageAccess",
  "Effect": "Allow",
  "Action": ["ecr:GetDownloadUrlForLayer", "ecr:BatchGetImage"],
  "Resource": "arn:aws:ecr:<ecrRegion>:<ecrAccountId>:repository/<ecrRepoName>"
},
{
  "Sid": "ECRTokenAccess",
  "Effect": "Allow",
  "Action": "ecr:GetAuthorizationToken",
  "Resource": "*"
}
```

### Step 4 — Mount persistent storage

Files written to a mount survive session termination and are visible to later invocations. Three types:

| Type | Scope | VPC? |
|---|---|---|
| Session storage | Service-managed, per-session. Persists across stop/resume for the same `runtimeSessionId`. | Not required |
| EFS access point | Your own EFS, shared across sessions, harnesses and agents. | Required |
| S3 Files access point | Your own S3 Files filesystem, syncing bidirectionally with the backing bucket. | Required |

Mount paths must be under `/mnt`. Start with session storage — no VPC needed:

```bash
aws bedrock-agentcore-control update-harness \
  --harness-id "$HARNESS_ID" \
  --environment '{"agentCoreRuntimeEnvironment": {"filesystemConfigurations": [{"sessionStorage": {"mountPath": "/mnt/data/"}}]}}'
```

*AgentCore CLI · new resources only*

```bash
# A new project, with session storage from the start
agentcore create --name myagent --session-storage-mount-path /mnt/data/

# A new harness inside an existing project
agentcore add harness --name storage_agent --session-storage /mnt/data/
agentcore deploy
```

> **Warning — `add harness` creates; it never updates**
>
> Reusing an existing name fails with `Error: Harness "x" already exists.` — there is no CLI verb that edits a deployed harness. To add session storage to the harness you have been building, edit its spec and redeploy:

*app/research_agent/harness.json*

```json
{
  "name": "research_agent",
  "model": { "provider": "bedrock", "modelId": "us.anthropic.claude-sonnet-4-6" },
  "sessionStoragePath": "/mnt/data/"
}
```

```bash
agentcore deploy
```

EFS and S3 Files both require VPC network mode. EFS:

```bash
aws bedrock-agentcore-control create-harness \
  --harness-name "SharedToolsAgent" \
  --execution-role-arn "$ROLE_ARN" \
  --environment '{
    "agentCoreRuntimeEnvironment": {
      "networkConfiguration": {
        "networkMode": "VPC",
        "networkModeConfig": {
          "subnets": ["subnet-abc123", "subnet-def456"],
          "securityGroups": ["sg-abc123"]
        }
      },
      "filesystemConfigurations": [
        {
          "efsAccessPoint": {
            "accessPointArn": "arn:aws:elasticfilesystem:us-west-2:123456789012:access-point/fsap-0123456789abcdef0",
            "mountPath": "/mnt/efs"
          }
        }
      ]
    }
  }'
```

*S3 Files, on the CLI*

```bash
agentcore add harness --name data_agent \
  --network-mode VPC \
  --subnets subnet-abc123,subnet-def456 \
  --security-groups sg-abc123 \
  --s3-access-point arn:aws:s3files:us-west-2:123456789012:file-system/fs-0123456789abcdef0/access-point/fsap-0123456789abcdef0:/mnt/s3data
agentcore deploy
```

The flags take `<accessPointArn>:<mountPath>`. Access point ARNs contain colons themselves, so the mount path is read from the segment after the *final* colon. Both `--efs-access-point` and `--s3-access-point` are repeatable, up to 2 mounts each.

> **Warning — `UpdateHarness` replaces the whole list**
>
> `filesystemConfigurations` is not merged. To add a mount to a harness that already has one, call `GetHarness` first and send the complete desired list — existing entries plus the new one.

> **Warning — VPC mode needs a NAT gateway**
>
> The harness pulls its application container from Amazon ECR Public at the start of every session. ECR Public does not support VPC endpoints, so your VPC must have a NAT gateway with a route to an internet gateway and allow outbound access to `public.ecr.aws`. Without it, sessions fail to start with image pull timeouts. The same egress is what Git skills and remote MCP servers need.

> **Checkpoint 07**
>
> You ran a shell command on the microVM with `--exec` and saw its stdout and exit code, and a file written by the agent in one invocation was still readable by a later `--exec` call on the same session ID.

## 08. Observability and cost controls

*~25 min · traces · logs · limits · CloudTrail · tags*

> **Goal**
>
> See every step the agent took in one place, then put hard caps on it so a runaway loop cannot burn through your budget.

### Step 1 — Read the traces

Every invocation generates traces, logs and metrics through AgentCore Observability in CloudWatch, with no extra configuration. Model calls, tool invocations, memory operations and shell commands each appear with timing and payload detail. Traces are available from the very first invocation — assuming you enabled Transaction Search in Lab 01.

```bash
# Stream logs. With one deployed runtime, omit the selector entirely.
agentcore logs

# Filter
agentcore logs --since 1h --level error

# List recent traces
agentcore traces list

# Inspect one
agentcore traces get <trace-id>

# Select explicitly when the project has more than one runtime
agentcore logs --runtime harness_researchagent_research_agent
```

> **Warning — `logs` and `traces` select by runtime, not by harness**
>
> These three commands take `--runtime`; there is no `--harness` flag on any of them, unlike `invoke`, `add tool` and `add skill`. The runtime backing a harness is named `harness_<project>_<harness>` — so project `researchagent` with harness `research_agent` gives `harness_researchagent_research_agent`. This is the same name you will see in `list-agent-runtimes` and in the CloudWatch log group path, and it is a reminder that a harness runs *inside* Runtime.

Or open the [AgentCore Observability dashboard](https://us-west-2.console.aws.amazon.com/cloudwatch/home?region=us-west-2#/gen-ai-observability/agent-core/agents) in CloudWatch. The value here is the unified view — one place that shows what the agent did across every capability, instead of stitching together separate log groups for the model, the browser, the code interpreter and memory.

Go back to a Lab 04 invocation that used both the browser and the code interpreter, and follow it end to end in a single trace.

### Step 2 — Understand what CloudTrail records

Harness operations are logged as management events (control plane) and data events (data plane). One thing will surprise you: harness resources appear under the resource type `AWS::BedrockAgentCore::Runtime`, not a harness-specific type, because the harness is a managed abstraction over Runtime.

| Plane | Event names |
|---|---|
| Management | `CreateHarness`, `UpdateHarness`, `DeleteHarness`, `GetHarness`, `ListHarnesses` |
| Data | `InvokeAgentRuntime`, `InvokeAgentRuntimeCommand` |

Note that data-plane operations appear under the underlying Runtime API names, not `InvokeHarness`. The `resources.ARN` field carries the harness ARN for control-plane events and the runtime ARN for data-plane events. Write your CloudTrail queries accordingly.

### Step 3 — Set hard caps

All of these are optional; omit them for service defaults.

| Limit | Caps | Default |
|---|---|---|
| `maxIterations` | Reasoning/action cycles per invocation | 75 |
| `timeoutSeconds` | Wall-clock time for one invocation | 3600 |
| `maxTokens` | Token budget per invocation | — |
| `idleRuntimeSessionTimeout` | How long an idle microVM stays warm | 900 |
| `maxLifetime` | Maximum lifetime of a microVM session | 28800 |

#### AgentCore CLI

```bash
agentcore add harness --name bounded_agent \
  --max-iterations 50 --timeout 1800 --max-tokens 8192 \
  --truncation-strategy sliding_window \
  --idle-timeout 600 --max-lifetime 14400
agentcore deploy

# Override for one call
agentcore invoke --harness bounded_agent --max-iterations 20 --harness-timeout 600 \
  "Quick lookup: what's the weather in Seattle?"
```

#### AWS CLI

```bash
aws bedrock-agentcore-control update-harness \
  --harness-id "$HARNESS_ID" \
  --max-iterations 50 \
  --timeout-seconds 1800 \
  --max-tokens 8192
```

Or pass `maxIterations`, `timeoutSeconds` or `maxTokens` directly in `invoke_harness` to override a single call.

### Exercise: make the agent hit a limit

Set `--max-iterations 2` and give it a task that genuinely needs more cycles. Confirm you get `stopReason: "max_iterations_exceeded"` rather than a hallucinated answer.

```bash
agentcore invoke --harness research_agent --max-iterations 2 --verbose \
  "Browse three separate sources, extract the population of each, and chart them."
```

> **Note — Runtime quotas apply too**
>
> Because the harness is backed by AgentCore Runtime, invocations are also subject to Runtime service quotas — not only the limits you set. Check both the harness and Runtime service quota pages when sizing for load.

### Step 4 — Tag for cost allocation

```bash
aws bedrock-agentcore-control create-harness \
  --harness-name "MyHarness" \
  --execution-role-arn "$ROLE_ARN" \
  --tags '{"team": "platform", "environment": "staging"}'
```

*or in harness.json*

```json
{
  "tags": {
    "team": "platform",
    "environment": "staging"
  }
}
```

Tags flow through to the deployed CloudFormation resources, so they show up in Cost Explorer against the underlying capabilities you are actually paying for.

> **Checkpoint 08**
>
> You followed one multi-tool invocation end to end in a single CloudWatch trace, and you deliberately triggered `max_iterations_exceeded` and saw it in the stream.

## 09. Versioning and endpoints

*~20 min · immutable versions · named endpoints · instant rollback*

> **Goal**
>
> Ship a change to production safely and roll it back in one call. Every harness update mints an immutable version; endpoints are the named pointers you move deliberately.

- Creating a harness automatically creates version 1.
- Every update creates a new version with a *complete, self-contained* configuration — model, prompt, tools, memory, limits, environment.
- Versions are immutable once created.
- The `DEFAULT` endpoint always follows the latest version. Named endpoints do not move until you move them.

### Step 1 — Trace the lifecycle

Read this table as a story: a change lands, `DEFAULT` follows it instantly, and `PROD` stays where you left it until you decide.

| Change | Version behaviour | Latest | Endpoints |
|---|---|---|---|
| Initial creation | Creates V1 automatically | V1 | DEFAULT → V1 |
| Model change | New version with the updated model | V2 | DEFAULT → V2 automatically |
| Create PROD endpoint at V2 | No new version | V2 | PROD → V2 |
| Tool or skill update | New version with updated tools | V3 | DEFAULT → V3; PROD stays V2 |
| Update PROD to V3 | No new version | V3 | PROD → V3 |
| Limits or environment change | New version with updated parameters | V4 | DEFAULT → V4; PROD stays V3 |

### Step 2 — Create a pinned endpoint

Omit `targetVersion` and the endpoint points at whatever is latest at creation time.

#### boto3

```python
import boto3

control_client = boto3.client('bedrock-agentcore-control', region_name='us-west-2')

response = control_client.create_harness_endpoint(
    harnessId='MyHarness-UuFdkQoXSL',
    endpointName='production-endpoint',
    targetVersion='2',
    description='Production endpoint pinned to V2'
)

print(response)
```

#### AWS CLI

```bash
aws bedrock-agentcore-control create-harness-endpoint \
  --harness-id "$HARNESS_ID" \
  --endpoint-name "production-endpoint" \
  --target-version "2" \
  --description "Production endpoint pinned to V2"
```

### Step 3 — Promote, then roll back

Make a change — swap the model, add a tool — which mints V3. `DEFAULT` moves; your endpoint does not. Then promote:

```bash
aws bedrock-agentcore-control update-harness-endpoint \
  --harness-id "$HARNESS_ID" \
  --endpoint-name "production-endpoint" \
  --target-version "3" \
  --description "Updated production endpoint"
```

```python
response = control_client.update_harness_endpoint(
    harnessId='MyHarness-UuFdkQoXSL',
    endpointName='production-endpoint',
    targetVersion='2',          # roll back by pointing at the earlier version
    description='Rolled back to V2'
)
```

That is the whole rollback story: point the endpoint at an earlier version. No redeploy, no rebuild — the old version still exists, immutably.

### Step 4 — Enumerate

```python
versions = control_client.list_harness_versions(harnessId='MyHarness-UuFdkQoXSL')
for version in versions['harnessVersions']:
    print(version)

endpoints = control_client.list_harness_endpoints(harnessId='MyHarness-UuFdkQoXSL')
for endpoint in endpoints['endpoints']:
    print(endpoint)
```

```bash
aws bedrock-agentcore-control list-harness-versions --harness-id "$HARNESS_ID"
aws bedrock-agentcore-control list-harness-endpoints --harness-id "$HARNESS_ID"
```

Both paginate through `maxResults` and `nextToken`. Use `GetHarnessEndpoint` for one endpoint's configuration and `DeleteHarnessEndpoint` to remove it.

### Endpoint lifecycle states

| State | Meaning |
|---|---|
| `CREATING` | Initial state while the endpoint is being created. |
| `CREATE_FAILED` | Creation failed — permissions, configuration, or another issue. |
| `READY` | Accepting requests. |
| `UPDATING` | Being updated to a new version. |
| `UPDATE_FAILED` | The update operation failed. |

> **Checkpoint 09**
>
> `list-harness-versions` shows at least three versions. You promoted a named endpoint to the newest one, invoked through it, then rolled it back — and confirmed the behaviour changed both times.

## 10. Security and access controls

*~30 min · IAM matrix · inbound OAuth · trust boundary · VPC*

> **Goal**
>
> Understand precisely where the security boundary sits, then move your harness from IAM auth to OAuth so end-user identity threads through to downstream tools.

### Step 1 — The permission matrix

Every harness API needs permissions on *both* the harness resource and the underlying Runtime resource. This trips people up constantly: granting `InvokeHarness` alone is not enough.

| API | Required IAM actions |
|---|---|
| `InvokeHarness` | `bedrock-agentcore:InvokeHarness`, `bedrock-agentcore:InvokeAgentRuntime` |
| `InvokeAgentRuntimeCommand` | `bedrock-agentcore:InvokeAgentRuntimeCommand`, `bedrock-agentcore:InvokeAgentRuntime` |
| `CreateHarness` | `bedrock-agentcore:CreateHarness`, `CreateAgentRuntime`, `CreateMemory` |
| `UpdateHarness` | `bedrock-agentcore:UpdateHarness`, `UpdateAgentRuntime`, `UpdateMemory` |
| `DeleteHarness` | `bedrock-agentcore:DeleteHarness`, `DeleteAgentRuntime`, `DeleteMemory` |
| `GetHarness` | `bedrock-agentcore:GetHarness` |
| `ListHarnesses` | `bedrock-agentcore:ListHarnesses` |
| `CreateHarnessEndpoint` | `bedrock-agentcore:CreateHarnessEndpoint`, `CreateAgentRuntimeEndpoint` |
| `UpdateHarnessEndpoint` | `bedrock-agentcore:UpdateHarnessEndpoint`, `UpdateAgentRuntimeEndpoint` |
| `DeleteHarnessEndpoint` | `bedrock-agentcore:DeleteHarnessEndpoint`, `DeleteAgentRuntimeEndpoint` |
| `GetHarnessEndpoint` | `bedrock-agentcore:GetHarnessEndpoint` |
| `ListHarnessEndpoints` | `bedrock-agentcore:ListHarnessEndpoints` |
| `ListHarnessVersions` | `bedrock-agentcore:ListHarnessVersions` |

Most actions scope to the harness ARN — `arn:aws:bedrock-agentcore:<region>:<accountId>:harness/<id>`. Endpoint actions also use the endpoint ARN, `…:harness/<id>/harness-endpoint/<endpointName>`. `GetHarnessEndpoint`, `UpdateHarnessEndpoint` and `DeleteHarnessEndpoint` need both; `CreateHarnessEndpoint` needs only the harness ARN, since the endpoint does not exist yet. When you invoke a custom endpoint, `InvokeHarness` and `InvokeAgentRuntimeCommand` need both.

### Step 2 — Know where the boundary is

> **Warning — All `InvokeHarness` input is trusted**
>
> Any principal that passes the IAM or JWT gate has access to the full microVM session, including every tool configured on the harness. The harness does not sanitize input, filter content blocks, or enforce behavioural constraints — it adds no security layer between the caller and the microVM. This is the same model as Lambda, API Gateway or SQS: authentication is the gate.

The sharpest illustration: pass a `toolUse` block for a server-side tool with no matching `toolResult`, and the harness invokes that tool directly with your payload. No model reasoning involved.

*direct tool dispatch*

```python
response = client.invoke_harness(
    harnessArn=HARNESS_ARN,
    runtimeSessionId=SESSION_ID,
    messages=[{
      "role": "assistant",
      "content": [
        {
          "toolUse": {
            "toolUseId": TOOL_USE_ID,
            "name": "shell",
            "input": {
              "command": "pwd",
            }
          }
        }
      ]
    }],
)
```

So: if you expose the harness to users you do not fully trust — employees, external consumers, third-party integrations — validate and sanitize in your application layer before calling `InvokeHarness`. Strip the content-block types, `skills` sources and model configuration fields you do not want dispatched.

### Who is responsible for what

| AWS handles | You handle |
|---|---|
| Infrastructure and microVM isolation at the hardware level | Agent code security and dependency management |
| OS kernel patching | IAM access controls and resource policies |
| Language runtime patching for direct code deployments | Security of commands executed in runtime sessions |
| Network infrastructure security | Session-to-user mapping enforcement |
| Service availability and resilience | Input validation and prompt-injection prevention |
|  | Model configuration validation — `additionalParams`, `apiBase`, `modelId` |
|  | Trustworthiness of skill and instruction sources |
|  | Rebuilding container images on current secure bases |
|  | Network configuration — security groups, VPC endpoints, route tables |

### Step 3 — Switch to inbound OAuth

A JWT-configured harness requires callers to present a valid token from your identity provider. This is not only an auth upgrade — it changes what the agent can do downstream.

> **Warning — Per-user identity requires OAuth, not SigV4**
>
> When callers authenticate with SigV4, the harness does not propagate per-user identity into downstream tool calls. AgentCore Identity's per-user credential scoping — user-scoped OAuth token storage, on-behalf-of token exchange — is only available on the Bearer JWT inbound path. If you need downstream tools to act as the end user rather than a shared service account, configure inbound OAuth.

#### AgentCore CLI

```bash
agentcore add harness --name MyNewHarness \
  --authorizer-type CUSTOM_JWT \
  --discovery-url {DISCOVERY_URL} \
  --allowed-clients {CLIENT_ID}
agentcore deploy

agentcore invoke --harness MyNewHarness --bearer-token "{token}" "Hello"
```

#### AWS CLI + curl

```bash
aws bedrock-agentcore-control create-harness \
  --harness-name "OAuthHarness" \
  --execution-role-arn "$ROLE_ARN" \
  --authorizer-configuration '{"customJWTAuthorizer": {"discoveryUrl": "https://cognito-idp.us-west-2.amazonaws.com/<POOL_ID>/.well-known/openid-configuration", "allowedClients": ["<CLIENT_ID>"]}}'
```

*invoke with a bearer token instead of SigV4*

```bash
curl -X POST "https://bedrock-agentcore.us-west-2.amazonaws.com/harnesses/invoke?harnessArn=${HARNESS_ARN}" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer ${ID_TOKEN}" \
  -H "X-Amzn-Bedrock-AgentCore-Runtime-Session-Id: $(uuidgen)" \
  -d '{"messages": [{"role": "user", "content": [{"text": "Hi"}]}]}'
```

If your IdP's OIDC discovery endpoint is only reachable over PrivateLink, add private-endpoint flags. Either a service-managed VPC endpoint:

```bash
agentcore add harness --name MyNewHarness \
  --authorizer-type CUSTOM_JWT \
  --discovery-url {DISCOVERY_URL} \
  --allowed-clients {CLIENT_ID} \
  --private-endpoint-vpc-id vpc-0abc1234def56789a \
  --private-endpoint-subnets subnet-0abc1234def56789a,subnet-0def5678abc12349b \
  --private-endpoint-ip-type IPV4 \
  --private-endpoint-security-groups sg-0abc1234def56789a
agentcore deploy
```

Or an existing VPC Lattice resource configuration: `--private-endpoint-lattice-arn rcfg-0abc1234def56789a`. These flags are valid only with `--authorizer-type CUSTOM_JWT`; `--private-endpoint-vpc-id` and `--private-endpoint-lattice-arn` are mutually exclusive, and the VPC form requires both `--private-endpoint-subnets` and `--private-endpoint-ip-type` (`IPV4` or `IPV6`).

### Step 4 — Put the harness in a VPC

By default harness sessions run on the public network. For private databases and internal APIs, move it into your VPC.

```bash
aws bedrock-agentcore-control create-harness \
  --harness-name "VpcHarness" \
  --execution-role-arn "$ROLE_ARN" \
  --environment '{"agentCoreRuntimeEnvironment": {"networkConfiguration": {"networkMode": "VPC", "vpcConfig": {"securityGroupIds": ["sg-0abc1234def56789a"], "subnetIds": ["subnet-0abc1234def56789a"]}}}}'
```

```bash
agentcore add harness --name internal_agent \
  --network-mode VPC \
  --subnets subnet-0abc1234def56789a \
  --security-groups sg-0abc1234def56789a
agentcore deploy
```

Remember the NAT gateway requirement from Lab 07 — ECR Public has no VPC endpoint, and without egress your sessions will not start.

### Step 5 — Layer policy on gateway tools

When tools come through AgentCore Gateway, Cedar-based policies gate every call: who may call which tool, under which conditions, with which arguments. This is the one place where you get true per-call authorization rather than a coarse allow/deny on the whole tool.

### Mitigations worth applying now

- Strip or allowlist the `model` field on caller-supplied requests; validate or remove `additionalParams`, `apiBase` and `modelId`.
- Deny `sts:AssumeRole` on the execution role unless role switching is genuinely required.
- Strip or ignore the `skills` field from untrusted requests; allowlist permitted S3 prefixes and Git repositories.
- Withhold `bedrock-agentcore:InvokeAgentRuntimeCommand` from principals who should not get a root shell.
- Scope network access with VPC security groups.
- Replace the Lab 01 wildcard on `BedrockModelInvocation` with specific inference profile ARNs.

> **Tip — On correlation IDs**
>
> The harness propagates correlation identifiers to Gateway, Memory, Code Interpreter and Browser so CloudWatch can show a unified trace. These are for observability only — never for authorization or data access decisions.

> **Checkpoint 10**
>
> You can name the two IAM actions `InvokeHarness` requires, explain in one sentence why `allowedTools` is not a defence against `InvokeAgentRuntimeCommand`, and say which inbound auth method is required for per-user downstream credentials.

## 11. Export the harness to code

*~20 min · Strands · CodeZip · the escape hatch*

> **Goal**
>
> Turn the declarative harness you have been building into editable Python you own. Export generates the equivalent agent using the Strands framework — a normal AgentCore Runtime agent you can modify freely and deploy anywhere.

Export when the configuration model runs out of room, not before. Three legitimate reasons:

- **Customise beyond the config** — add custom tool logic, change the agent loop, inject middleware or hooks, integrate libraries the harness does not expose.
- **Own the code** — move from declarative config to source you control, review and version alongside the rest of your application.
- **Graduate a prototype** — start fast with a harness, invest in a hand-maintained agent once the shape is settled.

If the configuration already does what you need, keep running it as a harness. Export is a one-way door in practice: you now maintain the loop.

### What comes across

Framework: Strands (Claude Agent SDK coming). Language: Python only. Build types: CodeZip (default) and Container. The generated agent mirrors the model provider and config including model ID and API key references; remote MCP tools, Gateways, inline function tools, Browser, Code Interpreter and the built-in `shell` and `file_operations`; memory; execution limits; conversation truncation; skills from path, S3 and Git; filesystem mounts for VPC harnesses; and authorizer configuration.

### Step 1 — Export

```bash
agentcore export harness --arn <arn> [options]
```

| Flag | Description | Default |
|---|---|---|
| `--arn <arn>` | Harness to export, when it was created outside the current CLI project. | `arn` or `name` required |
| `--name <name>` | Harness to export, when it lives in this CLI project. | `arn` or `name` required |
| `--target-agent-name` | Name for the generated runtime agent. | `<harnessName>Agent` |
| `--build <type>` | `CodeZip` or `Container`. | `CodeZip` |
| `--json` | Output results as JSON. | False |

```bash
# By ARN — generates MyHarnessAgent using CodeZip
agentcore export harness --arn arn:aws:bedrock-agentcore:us-east-1:123456789012:harness/MyHarness

# By project-local name
agentcore export harness --name MyHarness

# Custom agent name
agentcore export harness --name MyHarness --target-agent-name MyProductionAgent

# Container build — generates a Dockerfile
agentcore export harness --name MyHarness --build Container
```

Run `agentcore export harness` with no `--name` to get the interactive wizard: it asks which harness (skipped if there is only one), the generated agent's name, and the build type, then shows a confirmation summary.

### Step 2 — Read EXPORT_NOTES.md before deploying

> **Warning — This file is not optional reading**
>
> Every export writes `EXPORT_NOTES.md` into the agent directory listing each item that needs manual follow-up, why it needs attention, and exactly what to do about it. Read it before `agentcore deploy`.

```bash
cat EXPORT_NOTES.md
agentcore deploy
```

The exported agent is added as an ordinary runtime, so it deploys alongside the rest of your AgentCore CLI project.

### Step 3 — Consider where it runs now

Once it is code, hosting is your choice. `agentcore deploy` puts it on AgentCore Runtime with managed infrastructure, scaling and credential provisioning. Or self-host it anywhere Python 3.12+ runs: Lambda, ECS/Fargate, EC2, Kubernetes, on-premise servers, any containerised environment.

Compare the generated Python against your `harness.json` side by side. Everything that was one config line is now explicit construction — which is precisely the trade you just made.

> **Checkpoint 11**
>
> You exported the research agent, read `EXPORT_NOTES.md`, and can point to at least one thing in the generated code that was a single field in the harness configuration.

## 12. Harness or Runtime — and cleaning up

*~15 min · decision framework · resource teardown*

> **Goal**
>
> Decide correctly for your next project, then delete everything this lab created.

**AgentCore Runtime** is a serverless hosting environment. You bring agent code in any framework or none, wrap it with the SDK's `BedrockAgentCoreApp` entrypoint, package it into an ARM64 container, push to ECR and deploy. *The orchestration loop is yours.* Every other AgentCore primitive — Memory, Gateway, Browser, Code Interpreter, outbound Identity — you call from your code.

**AgentCore harness** provides the loop, powered by Strands Agents. You declare what the agent is; AWS runs it. It is a managed abstraction that runs *inside* Runtime — which is why CloudTrail files it under `AWS::BedrockAgentCore::Runtime`.

### Where they genuinely differ

Nearly every capability exists on both sides; the difference is whether you write code for it. The rows below are the ones that should actually decide your choice — everything the harness marks ✅ is available on Runtime too, but only as code you maintain.

| Capability | Harness | Runtime |
|---|---|---|
| Choice of agent framework | ❌ Strands only | 🔵 Any — your code |
| Bidirectional streaming | ❌ | 🔵 Your code |
| Non-agent-loop patterns (graph, workflow) | ❌ | 🔵 Your code |
| Hooks | ❌ | 🔵 Your code |
| Inline / client-side tools | 🔵 Yes, you write the handler | 🔵 Your code |
| Custom container image | 🟣 Config enables it, code may be needed | 🟣 Same |
| Model switching mid-session | ✅ No code | 🔵 Your code |
| Skills, memory, tools, limits, truncation | ✅ No code | 🔵 Your code |
| Streaming responses | ✅ No code | 🔵 Your code |
| Outbound auth / Identity token vault | ✅ No code | 🔵 Your code |
| Filesystem, env vars, VPC, inbound auth, versioning, exec API | ✅ No code | ✅ No code |

Legend: ✅ supported with no custom code · 🔵 supported, you maintain the implementation · 🟣 configuration enables it but code is required to fully use it · ❌ not supported.

Read it as a single question: *do you need to own the loop?* If you need a framework other than Strands, bidirectional streaming, a graph or workflow topology rather than an agent loop, or lifecycle hooks — use Runtime. Otherwise the harness gives you the same capabilities without the code to maintain, and Lab 11 is your exit if that changes.

### Clean up

What follows tears down one pass of this lab. If you intend to *run the lab again*, use [Reset & rerun](#reset-the-account-and-run-the-lab-again) instead — it is idempotent, scoped so it cannot touch unrelated resources, and covers the account-level setup (model agreement, Transaction Search, CDK bootstrap) that this section leaves in place.

> **Warning — Deleting a harness cascade-deletes its managed memory**
>
> By default `DeleteHarness` also deletes the memory it provisioned, and with it every stored conversation. Pass `deleteManagedMemory=false` if you want to keep the memory as a standalone resource instead.

#### AgentCore CLI

```bash
# The CLI manages a CloudFormation stack — tear it down.
# There is no `agentcore destroy`; delete the stack directly.
aws cloudformation delete-stack \
  --region "$AWS_REGION" \
  --stack-name "AgentCore-<project-name>-default"

aws cloudformation wait stack-delete-complete \
  --region "$AWS_REGION" \
  --stack-name "AgentCore-<project-name>-default"

# Confirm: both should come back empty
aws bedrock-agentcore-control list-agent-runtimes \
  --region "$AWS_REGION" --query 'agentRuntimes[].agentRuntimeName' --output text
```

The stack owns the harness, its runtime and the execution role the CLI generated (`<project>_<harness>`) — deleting it removes all three. Two things it does *not* remove: the runtime's CloudWatch log group under `/aws/bedrock-agentcore/runtimes/`, and the CDK bootstrap stack `CDKToolkit` that the first `agentcore deploy` created implicitly. Leave `CDKToolkit` in place unless you want a bare account; if you do delete it, empty its versioned asset bucket (`cdk-hnb659fds-assets-<account>-<region>`, including delete markers) and force-delete its ECR repo first, or the stack deletion fails.

#### AWS CLI

```bash
# 1. Endpoints first
for EP in $(aws bedrock-agentcore-control list-harness-endpoints \
              --harness-id "$HARNESS_ID" \
              --query 'endpoints[?name!=`DEFAULT`].name' --output text); do
  aws bedrock-agentcore-control delete-harness-endpoint \
    --harness-id "$HARNESS_ID" --endpoint-name "$EP"
done

# 2. Each harness you created
aws bedrock-agentcore-control delete-harness --harness-id "$HARNESS_ID"

# 3. Credential providers
aws bedrock-agentcore-control delete-api-key-credential-provider --name my-openai-key

# 4. Lab bucket
aws s3 rb "s3://${SKILL_BUCKET}" --force

# 5. IAM role
aws iam delete-role-policy --role-name "$ROLE_NAME" --policy-name HarnessBaseline
aws iam delete-role --role-name "$ROLE_NAME"

# 6. Runtime log groups (not removed with the harness)
aws logs delete-log-group \
  --log-group-name "/aws/bedrock-agentcore/runtimes/<runtime-id>-DEFAULT"

# 7. Model agreement — only if you want to undo Lab 01 Step 5.
#    Leave it if you plan to use the model again; it costs nothing idle.
aws bedrock delete-foundation-model-agreement --model-id "$MODEL_ID"

# 8. Transaction Search, to undo Lab 01 Step 4 (account-level setting)
aws xray update-trace-segment-destination --destination XRay
```

List anything you may have forgotten with `aws bedrock-agentcore-control list-harnesses`. Also check for gateways, custom browsers or code interpreters, EFS/S3 Files access points, and any ECR repositories created in Lab 07.

> **Checkpoint 12**
>
> `list-harnesses` returns empty, the lab bucket and IAM role are gone, and you can state in one sentence when you would choose Runtime over the harness.

## Reset the account and run the lab again

*~10 min · idempotent teardown · safe to run at any point*

> **Goal**
>
> Return the account to its pre-Lab-01 state so the whole lab — or any prefix of it — can be run again from a clean slate. Lab 12's cleanup is the *end* of a single pass; this section is the one you come back to. Every command here is safe to run when the resource is already absent, so you can run the block top to bottom without checking first.

Decide how far back you want to go. The full reset exercises Lab 01 Step 5 and the CDK bootstrap again, at the cost of a few extra minutes on the next pass.

| Reset | Removes | Cost on the next pass |
|---|---|---|
| **Fast** — steps 1–4 | Harness, runtime, roles, log groups, local project | None. Model access and CDK bootstrap are still in place. |
| **Full** — steps 1–6 | Also the model agreement, Transaction Search and CDK bootstrap | ~3 min for the agreement to propagate, plus a slower first `agentcore deploy` while CDK re-bootstraps. |

### Step 0 — Re-establish your shell

You are likely in a new terminal since the last pass. These are the same variables from Lab 01, plus the project name you gave `agentcore create`.

```bash
export AWS_REGION=us-west-2
export AWS_DEFAULT_REGION=us-west-2
export ACCOUNT_ID="$(aws sts get-caller-identity --query Account --output text)"
export ROLE_NAME=AgentCoreHarnessLabRole
export PROJECT=researchagent
export MODEL_ID=anthropic.claude-sonnet-4-6
export LAB_DIR="$HOME/PycharmProjects/aws_agent_harness"   # adjust to taste
```

### Step 1 — See what is actually there

Always look before deleting. This is also the fastest way to spot resources a partial pass left behind.

```bash
aws cloudformation list-stacks \
  --stack-status-filter CREATE_COMPLETE UPDATE_COMPLETE ROLLBACK_COMPLETE \
  --query 'StackSummaries[].StackName' --output text

aws bedrock-agentcore-control list-agent-runtimes \
  --query 'agentRuntimes[].agentRuntimeName' --output text

aws iam list-roles \
  --query "Roles[?contains(RoleName,'${PROJECT}')||RoleName=='${ROLE_NAME}'].RoleName" \
  --output text
```

### Step 2 — Delete the CloudFormation stack

The stack created by `agentcore deploy` owns the harness, its runtime, and the execution role the CLI generated (`<project>_<harness>`). Deleting the stack removes all three — do not delete them individually first, or the stack deletion will fail on resources that no longer exist.

```bash
STACK="AgentCore-${PROJECT}-default"

aws cloudformation delete-stack --stack-name "$STACK"
aws cloudformation wait stack-delete-complete --stack-name "$STACK"

# Should be empty
aws bedrock-agentcore-control list-agent-runtimes \
  --query 'agentRuntimes[].agentRuntimeName' --output text
```

> **Warning — There is no `agentcore destroy`**
>
> The CLI (0.24.x) has no teardown verb — `agentcore --help` lists `deploy` but nothing that reverses it. Delete the stack directly, as above. `agentcore remove` is unrelated: it edits `agentcore.json` and touches nothing in AWS.

### Step 3 — Delete the Lab 01 role and the log groups

An IAM role cannot be deleted while it holds policies. The log groups outlive the runtime that wrote them, and a stale one will not block a rerun — but it will leave you reading last pass's traces in Lab 08.

```bash
# Inline policies first, then the role
aws iam delete-role-policy \
  --role-name "$ROLE_NAME" --policy-name HarnessBaseline 2>/dev/null
aws iam delete-role --role-name "$ROLE_NAME" 2>/dev/null

# Log groups, scoped by project prefix so nothing unrelated is caught
for LG in $(aws logs describe-log-groups \
    --log-group-name-prefix "/aws/bedrock-agentcore/runtimes/harness_${PROJECT}_" \
    --query 'logGroups[].logGroupName' --output text); do
  echo "deleting $LG"
  aws logs delete-log-group --log-group-name "$LG"
done
```

> **Warning — Never delete log groups by wildcard**
>
> The prefix above is deliberately narrow. `/aws/bedrock-agentcore/runtimes/` is shared by *every* agent runtime in the account, including ones this lab did not create. Filtering on `harness_${PROJECT}_` keeps the blast radius to this project. Read the `echo` output before you trust the loop.

### Step 4 — Remove the local project

`agentcore create` will not scaffold over an existing directory, so this is required for a clean rerun. Check for your own work first — the scaffold's own commit is titled `Initial commit` and the project has no remote.

```bash
cd "$LAB_DIR"                      # NOT inside $PROJECT — see below
git -C "$PROJECT" log --oneline    # anything of yours in here?

rm -rf "$PROJECT" trust-policy.json harness-policy.json
```

> **Warning — Leave the project directory before deleting it**
>
> The `cd "$LAB_DIR"` above is load-bearing. Deleting the directory your shell is sitting in leaves it with a working directory that no longer exists, and every subsequent Node-based command dies before it starts:
>
> `Error: ENOENT: no such file or directory, uv_cwd — at process.wrappedCwd`
>
> It looks like a broken `npm` or a corrupted Node install; it is neither. `cd` anywhere that exists and it clears immediately. Python and the AWS CLI fail the same way with a less recognisable message.

**Stop here for a fast reset.** Everything below undoes account-level setup that costs nothing to keep and takes minutes to recreate. Continue only if you want Lab 01 Steps 4–5 to run against a genuinely fresh account.

### Step 5 — Undo model access and Transaction Search

Deleting the agreement puts `agreementAvailability` back to `NOT_AVAILABLE`, which is what makes Lab 01 Step 5 meaningful on the next pass rather than a no-op.

```bash
aws bedrock delete-foundation-model-agreement --model-id "$MODEL_ID"

# Confirm — expect NOT_AVAILABLE
aws bedrock get-foundation-model-availability \
  --model-id "$MODEL_ID" --query 'agreementAvailability.status' --output text

# Transaction Search back to the default destination (undoes Lab 01 Step 4)
aws xray update-trace-segment-destination --destination XRay
```

> **Note — Why bother resetting Transaction Search**
>
> Leaving it enabled makes Lab 01 Step 4 fail with `InvalidRequestException: The destination is already set to CloudWatchLogs`. Harmless, but it reads like a real error mid-run. Note this is an *account-level* setting: if other agents in this account rely on trace indexing, leave it alone and expect that message instead.

### Step 6 — Remove the CDK bootstrap

The first `agentcore deploy` bootstraps CDK implicitly, creating the `CDKToolkit` stack, a versioned S3 asset bucket and an ECR repository. No lab step creates these and keeping them is free, so this step is genuinely optional. The ordering matters — the stack will not delete while the bucket has contents.

```bash
BUCKET="cdk-hnb659fds-assets-${ACCOUNT_ID}-${AWS_REGION}"

# 1. Current objects
aws s3 rm "s3://${BUCKET}" --recursive

# 2. Versions AND delete markers — the bucket is versioned, so step 1
#    leaves both behind and the bucket is still not empty
while :; do
  aws s3api list-object-versions --bucket "$BUCKET" --max-keys 500 \
    --query '{Objects: [Versions, DeleteMarkers][][].{Key:Key,VersionId:VersionId}}' \
    --output json > /tmp/cdk-purge.json
  COUNT=$(python3 -c 'import json,sys; print(len(json.load(open("/tmp/cdk-purge.json")).get("Objects") or []))')
  echo "remaining: $COUNT"
  [ "$COUNT" = "0" ] && break
  aws s3api delete-objects --bucket "$BUCKET" --delete file:///tmp/cdk-purge.json >/dev/null
done

# 3. ECR repository, images and all
aws ecr delete-repository --force \
  --repository-name "cdk-hnb659fds-container-assets-${ACCOUNT_ID}-${AWS_REGION}"

# 4. The stack
aws cloudformation delete-stack --stack-name CDKToolkit
aws cloudformation wait stack-delete-complete --stack-name CDKToolkit

# 5. The bucket survives the stack — its deletion policy is RETAIN
aws s3api delete-bucket --bucket "$BUCKET"
```

> **Warning — Two traps in step 6**
>
> `aws s3 rm --recursive` does *not* empty a versioned bucket — it writes delete markers over the current versions and leaves every prior version in place. Skipping the purge loop gives you a `BucketNotEmpty` failure several minutes into the stack deletion. And the bucket carries a `RETAIN` deletion policy, so it outlives `CDKToolkit`; without the final `delete-bucket` it lingers, and a later re-bootstrap in the same region will adopt it.

### Verify

Run this after either reset. A fast reset leaves the agreement `AVAILABLE` and `CDKToolkit` listed; a full reset returns empty for everything except the service-linked role.

```bash
echo "stacks:    $(aws cloudformation list-stacks \
  --stack-status-filter CREATE_COMPLETE UPDATE_COMPLETE \
  --query 'StackSummaries[].StackName' --output text)"
echo "runtimes:  $(aws bedrock-agentcore-control list-agent-runtimes \
  --query 'agentRuntimes[].agentRuntimeName' --output text)"
echo "roles:     $(aws iam list-roles \
  --query "Roles[?contains(RoleName,'${PROJECT}')||RoleName=='${ROLE_NAME}'].RoleName" --output text)"
echo "agreement: $(aws bedrock get-foundation-model-availability \
  --model-id "$MODEL_ID" --query 'agreementAvailability.status' --output text)"
echo "logs:      $(aws logs describe-log-groups \
  --log-group-name-prefix "/aws/bedrock-agentcore/runtimes/harness_${PROJECT}_" \
  --query 'logGroups[].logGroupName' --output text)"
echo "buckets:   $(aws s3api list-buckets \
  --query 'Buckets[?contains(Name,`cdk-hnb659fds`)].Name' --output text)"
```

> **Note — What is left behind on purpose**
>
> `AWSServiceRoleForBedrockAgentCoreRuntimeIdentity` is an AWS service-linked role. It is recreated automatically the first time you deploy, so deleting it accomplishes nothing. Any log group not matching your project prefix belongs to other work in this account and is not the lab's to remove.

> **Note — You are reset when**
>
> The verify block reports empty `stacks`, `runtimes`, `roles` and `logs`, and `$LAB_DIR` no longer contains the project directory. If you ran the full reset, `agreement` reads `NOT_AVAILABLE` and `buckets` is empty. Clear your ticks with **Reset progress** at the top of the page, then start again at Lab 01 Step 1.

## Reference

*API surface · defaults · things worth memorising*

### API surface

Control plane on `bedrock-agentcore-control`; data plane on `bedrock-agentcore`.

| Plane | Operations |
|---|---|
| Control — harness | `CreateHarness` · `GetHarness` · `UpdateHarness` · `DeleteHarness` · `ListHarnesses` · `ListHarnessVersions` |
| Control — endpoints | `CreateHarnessEndpoint` · `GetHarnessEndpoint` · `UpdateHarnessEndpoint` · `DeleteHarnessEndpoint` · `ListHarnessEndpoints` |
| Data | `InvokeHarness` · `InvokeAgentRuntimeCommand` |

### Defaults worth remembering

| Thing | Default |
|---|---|
| Model | Claude Sonnet 4.6 on Bedrock (`global.anthropic.claude-sonnet-4-6`) |
| Model API format | `converse_stream` (Bedrock) · `responses` (OpenAI) |
| Memory | Provisioned automatically: `SEMANTIC` + `SUMMARIZATION`, 30-day event expiry |
| Memory retrieval | `topK=10`, `relevanceScore=0.2`, derived per active strategy |
| Truncation | `sliding_window` |
| Tools present | `shell` and `file_operations`, unless restricted by `allowedTools` |
| Network | Public. VPC is opt-in. |
| `maxIterations` / `timeoutSeconds` | 75 / 3600 s |
| `idleRuntimeSessionTimeout` / `maxLifetime` | 900 s / 28800 s |
| Gateway outbound auth | SigV4 (AWS IAM) |
| Git skill username | `oauth2` |

### Hard constraints

- `runtimeSessionId` must be at least 33 characters.
- Container images must be `linux/arm64`; the harness overrides `ENTRYPOINT` and `CMD`.
- Filesystem mount paths must be under `/mnt`. Up to 2 EFS and 2 S3 Files mounts.
- `UpdateHarness` replaces `filesystemConfigurations` wholesale — read first, then send the full list.
- Git skill clones must finish in 60 s; each S3 skill must be ≤ 1 GB.
- AWS skill paths must be relative — no leading `/`, no `..`.
- VPC mode requires a NAT gateway for `public.ecr.aws`; ECR Public has no VPC endpoint.
- Web Search connector: `us-east-1` only.
- Guardrails require the `converse_stream` API format.

### Source pages

Every command and policy in this lab comes from the AgentCore developer guide harness chapters: [overview](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/harness.html), [get started](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/harness-get-started.html), [models](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/harness-models.html), [tools](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/harness-tools.html), [skills](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/harness-skills.html), [memory](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/harness-memory.html), [environment](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/harness-environment.html), [operations](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/harness-operations.html), [versioning](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/harness-versioning.html), [export](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/harness-export.html), [security](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/harness-security.html), [harness vs. Runtime](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/harness-vs-runtime.html).
