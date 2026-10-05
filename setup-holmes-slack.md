# HolmesGPT in Kubernetes, answering in Slack

HolmesGPT is an AI agent that investigates problems in a Kubernetes cluster. This guide runs it inside your cluster, connects it to an AI model, and lets you talk to it from Slack. When it finds a fix, it asks for your approval before changing anything.

The setup has five parts:

1. A Kubernetes cluster (kind, on your laptop)
2. Broken practice apps for Holmes to investigate
3. HolmesGPT running inside the cluster, using an AI model (an OpenAI-compatible endpoint, or Gemini)
4. A Slack app
5. A small custom bot that connects Slack to Holmes, with Approve and Reject buttons

Everything runs inside your cluster. Slack never needs to reach your machine, because the bot opens the connection out to Slack (Socket Mode). 

```
You in Slack ──@holmes──▶ Slack ══(Socket Mode, opened by the bot)══▶ bot pod
                                                                        │ POST /api/chat
                                                                        ▼
                                Slack thread ◀── bot ◀── Holmes ──▶ AI model + kubectl
```

Tested with the HolmesGPT Helm chart `0.42.0`, kind, Kubernetes `v1.36`, and both LM Studio (`qwen/qwen3.5-9b`) and Gemini (`gemini-3.1-flash-lite`).

---

## 0. What you need

| Tool | Check | Install (macOS) |
|---|---|---|
| Docker Desktop | `docker info` | docker.com |
| kind | `kind version` | `brew install kind` |
| kubectl | `kubectl version --client` | `brew install kubectl` |
| Helm | `helm version` | `brew install helm` |
| jq | `jq --version` | `brew install jq` |

You also need:
- **An AI model**, either an OpenAI-compatible endpoint (LM Studio, vLLM, LiteLLM, Ollama, OpenAI) with its URL and key, or a **Gemini API key** from [aistudio.google.com/apikey](https://aistudio.google.com/apikey).
- **A Slack workspace** where you're allowed to create apps.

---

## 1. Create the cluster

```bash
kind create cluster --name sre --wait 120s
```
```bash
kubectl config use-context kind-sre
```
```bash
kubectl get nodes
```
You should see one node, `sre-control-plane`, with status `Ready`.

---

## 2. Create the practice scenarios

Each scenario gets its own namespace and is broken in one specific way. Run the steps one at a time, then wait about a minute and check with `kubectl get pods -A`.

### 2.1 Healthy baseline (`sre-healthy`)
nginx, Redis and Postgres all working. Holmes should find **nothing** wrong here, which makes it a good control.
```bash
kubectl apply -f - <<'EOF'
apiVersion: v1
kind: Namespace
metadata: { name: sre-healthy }
---
apiVersion: apps/v1
kind: Deployment
metadata: { name: web, namespace: sre-healthy }
spec:
  replicas: 2
  selector: { matchLabels: { app: web } }
  template:
    metadata: { labels: { app: web } }
    spec:
      containers:
        - name: nginx
          image: nginx:1.27
          ports: [{ containerPort: 80 }]
          resources: { requests: { cpu: 10m, memory: 32Mi }, limits: { memory: 128Mi } }
          readinessProbe: { httpGet: { path: /, port: 80 } }
---
apiVersion: v1
kind: Service
metadata: { name: web, namespace: sre-healthy }
spec: { selector: { app: web }, ports: [{ port: 80, targetPort: 80 }] }
---
apiVersion: apps/v1
kind: Deployment
metadata: { name: redis, namespace: sre-healthy }
spec:
  selector: { matchLabels: { app: redis } }
  template:
    metadata: { labels: { app: redis } }
    spec:
      containers:
        - name: redis
          image: redis:7.4
          ports: [{ containerPort: 6379 }]
          resources: { requests: { cpu: 10m, memory: 32Mi }, limits: { memory: 128Mi } }
---
apiVersion: v1
kind: Service
metadata: { name: redis, namespace: sre-healthy }
spec: { selector: { app: redis }, ports: [{ port: 6379 }] }
---
apiVersion: v1
kind: Secret
metadata: { name: postgres, namespace: sre-healthy }
stringData: { POSTGRES_PASSWORD: practice-only }
---
apiVersion: apps/v1
kind: StatefulSet
metadata: { name: postgres, namespace: sre-healthy }
spec:
  serviceName: postgres
  selector: { matchLabels: { app: postgres } }
  template:
    metadata: { labels: { app: postgres } }
    spec:
      containers:
        - name: postgres
          image: postgres:17
          envFrom: [{ secretRef: { name: postgres } }]
          ports: [{ containerPort: 5432 }]
          resources: { requests: { cpu: 50m, memory: 128Mi }, limits: { memory: 512Mi } }
          volumeMounts: [{ name: data, mountPath: /var/lib/postgresql/data }]
  volumeClaimTemplates:
    - metadata: { name: data }
      spec: { accessModes: [ReadWriteOnce], resources: { requests: { storage: 1Gi } } }
---
apiVersion: v1
kind: Service
metadata: { name: postgres, namespace: sre-healthy }
spec: { selector: { app: postgres }, ports: [{ port: 5432 }] }
EOF
```

### 2.2 Image that doesn't exist (`sre-imagepull`)
**Symptom:** `ImagePullBackOff`. **Cause:** the tag `nginx:1.27-doesnotexist` doesn't exist.
```bash
kubectl apply -f - <<'EOF'
apiVersion: v1
kind: Namespace
metadata: { name: sre-imagepull }
---
apiVersion: apps/v1
kind: Deployment
metadata: { name: web, namespace: sre-imagepull }
spec:
  selector: { matchLabels: { app: web } }
  template:
    metadata: { labels: { app: web } }
    spec:
      containers:
        - name: nginx
          image: nginx:1.27-doesnotexist
EOF
```

### 2.3 App that crashes on start (`sre-crashloop`)
**Symptom:** `CrashLoopBackOff`. **Cause:** Postgres refuses to start without `POSTGRES_PASSWORD`. The reason is only in the logs.
```bash
kubectl apply -f - <<'EOF'
apiVersion: v1
kind: Namespace
metadata: { name: sre-crashloop }
---
apiVersion: apps/v1
kind: Deployment
metadata: { name: postgres, namespace: sre-crashloop }
spec:
  selector: { matchLabels: { app: postgres } }
  template:
    metadata: { labels: { app: postgres } }
    spec:
      containers:
        - name: postgres
          image: postgres:17
EOF
```

### 2.4 Out of memory (`sre-oom`)
**Symptom:** `OOMKilled`, exit code 137. **Cause:** the command `tail /dev/zero` uses memory without end, so it hits the 32Mi limit. Raising the limit doesn't fix it, which makes this a good test of whether the AI's fix is actually right.
```bash
kubectl apply -f - <<'EOF'
apiVersion: v1
kind: Namespace
metadata: { name: sre-oom }
---
apiVersion: apps/v1
kind: Deployment
metadata: { name: report-worker, namespace: sre-oom }
spec:
  selector: { matchLabels: { app: report-worker } }
  template:
    metadata: { labels: { app: report-worker } }
    spec:
      containers:
        - name: worker
          image: busybox:1.37
          command: ["sh", "-c", "echo building report cache; tail /dev/zero"]
          resources: { requests: { memory: 16Mi }, limits: { memory: 32Mi } }
EOF
```

### 2.5 Pods that never get scheduled (`sre-pending`)
**Symptom:** two pods stuck in `Pending`. **Causes:** `analytics` asks for 64 CPUs, more than any node has, and `mysql` needs a PVC that asks for a StorageClass `fast-ssd` that doesn't exist (kind only has `standard`).
```bash
kubectl apply -f - <<'EOF'
apiVersion: v1
kind: Namespace
metadata: { name: sre-pending }
---
apiVersion: apps/v1
kind: Deployment
metadata: { name: analytics, namespace: sre-pending }
spec:
  selector: { matchLabels: { app: analytics } }
  template:
    metadata: { labels: { app: analytics } }
    spec:
      containers:
        - name: redis
          image: redis:7.4
          resources: { requests: { cpu: "64", memory: 64Mi } }
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata: { name: mysql-data, namespace: sre-pending }
spec:
  storageClassName: fast-ssd
  accessModes: [ReadWriteOnce]
  resources: { requests: { storage: 1Gi } }
---
apiVersion: apps/v1
kind: Deployment
metadata: { name: mysql, namespace: sre-pending }
spec:
  selector: { matchLabels: { app: mysql } }
  template:
    metadata: { labels: { app: mysql } }
    spec:
      containers:
        - name: mysql
          image: mysql:8.4
          env: [{ name: MYSQL_ROOT_PASSWORD, value: practice-only }]
          volumeMounts: [{ name: data, mountPath: /var/lib/mysql }]
      volumes: [{ name: data, persistentVolumeClaim: { claimName: mysql-data } }]
EOF
```

### 2.6 Service with no endpoints (`sre-service`)
**Symptom:** the pod is `Running`, but the Service sends traffic nowhere. **Causes:** the Service selector says `app: shopp` while the pods are labeled `app: shop`, and the Ingress points at a Service `shop-frontend` that doesn't exist. **This is the best scenario for testing approved fixes**: the fix is one `kubectl patch`.
```bash
kubectl apply -f - <<'EOF'
apiVersion: v1
kind: Namespace
metadata: { name: sre-service }
---
apiVersion: apps/v1
kind: Deployment
metadata: { name: shop, namespace: sre-service }
spec:
  selector: { matchLabels: { app: shop } }
  template:
    metadata: { labels: { app: shop } }
    spec:
      containers:
        - name: nginx
          image: nginx:1.27
          ports: [{ containerPort: 80 }]
---
apiVersion: v1
kind: Service
metadata: { name: shop, namespace: sre-service }
spec: { selector: { app: shopp }, ports: [{ port: 80, targetPort: 80 }] }
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata: { name: shop, namespace: sre-service }
spec:
  rules:
    - host: shop.local
      http:
        paths:
          - path: /
            pathType: Prefix
            backend: { service: { name: shop-frontend, port: { number: 80 } } }
EOF
```

### 2.7 Wrong health checks (`sre-probes`)
**Symptoms:** `api` is `Running` but never `Ready` (0/1), and `gateway` keeps restarting. **Causes:** the readiness check uses port 8080 while nginx listens on 80, and the liveness check calls `/healthz`, which returns 404.
```bash
kubectl apply -f - <<'EOF'
apiVersion: v1
kind: Namespace
metadata: { name: sre-probes }
---
apiVersion: apps/v1
kind: Deployment
metadata: { name: api, namespace: sre-probes }
spec:
  selector: { matchLabels: { app: api } }
  template:
    metadata: { labels: { app: api } }
    spec:
      containers:
        - name: nginx
          image: nginx:1.27
          ports: [{ containerPort: 80 }]
          readinessProbe: { httpGet: { path: /, port: 8080 }, periodSeconds: 5 }
---
apiVersion: apps/v1
kind: Deployment
metadata: { name: gateway, namespace: sre-probes }
spec:
  selector: { matchLabels: { app: gateway } }
  template:
    metadata: { labels: { app: gateway } }
    spec:
      containers:
        - name: nginx
          image: nginx:1.27
          ports: [{ containerPort: 80 }]
          livenessProbe: { httpGet: { path: /healthz, port: 80 }, initialDelaySeconds: 5, periodSeconds: 5 }
EOF
```

### 2.8 Missing Secret and ConfigMap (`sre-config`)
**Symptoms:** `mysql` shows `CreateContainerConfigError`, and `web` is stuck in `ContainerCreating`. **Causes:** the Secret `mysql-credentials` and the ConfigMap `nginx-site-config` were never created.
```bash
kubectl apply -f - <<'EOF'
apiVersion: v1
kind: Namespace
metadata: { name: sre-config }
---
apiVersion: apps/v1
kind: Deployment
metadata: { name: mysql, namespace: sre-config }
spec:
  selector: { matchLabels: { app: mysql } }
  template:
    metadata: { labels: { app: mysql } }
    spec:
      containers:
        - name: mysql
          image: mysql:8.4
          env:
            - name: MYSQL_ROOT_PASSWORD
              valueFrom: { secretKeyRef: { name: mysql-credentials, key: root-password } }
---
apiVersion: apps/v1
kind: Deployment
metadata: { name: web, namespace: sre-config }
spec:
  selector: { matchLabels: { app: web } }
  template:
    metadata: { labels: { app: web } }
    spec:
      containers:
        - name: nginx
          image: nginx:1.27
          volumeMounts: [{ name: conf, mountPath: /etc/nginx/conf.d }]
      volumes: [{ name: conf, configMap: { name: nginx-site-config } }]
EOF
```

### 2.9 Wrong hostname for a dependency (`sre-dependency`)
**Symptom:** `checkout` crashes, while Redis is perfectly healthy. **Cause:** the app looks for `redis-cache`, but the Service is called `redis`. Only the logs show this (`cannot connect to session store redis-cache:6379`), so it's the hardest test and the one that best shows what Holmes can do.
```bash
kubectl apply -f - <<'EOF'
apiVersion: v1
kind: Namespace
metadata: { name: sre-dependency }
---
apiVersion: apps/v1
kind: Deployment
metadata: { name: redis, namespace: sre-dependency }
spec:
  selector: { matchLabels: { app: redis } }
  template:
    metadata: { labels: { app: redis } }
    spec:
      containers:
        - name: redis
          image: redis:7.4
          ports: [{ containerPort: 6379 }]
---
apiVersion: v1
kind: Service
metadata: { name: redis, namespace: sre-dependency }
spec: { selector: { app: redis }, ports: [{ port: 6379 }] }
---
apiVersion: v1
kind: ConfigMap
metadata: { name: checkout-config, namespace: sre-dependency }
data: { REDIS_HOST: redis-cache, REDIS_PORT: "6379" }
---
apiVersion: apps/v1
kind: Deployment
metadata: { name: checkout, namespace: sre-dependency }
spec:
  selector: { matchLabels: { app: checkout } }
  template:
    metadata: { labels: { app: checkout } }
    spec:
      containers:
        - name: checkout
          image: redis:7.4
          envFrom: [{ configMapRef: { name: checkout-config } }]
          command: ["sh", "-c"]
          args:
            - |
              echo "checkout service starting, session store at $REDIS_HOST:$REDIS_PORT"
              for i in 1 2 3; do
                redis-cli -h "$REDIS_HOST" -p "$REDIS_PORT" ping && exec sleep infinity
                echo "ERROR: cannot connect to session store $REDIS_HOST:$REDIS_PORT (attempt $i)"
                sleep 3
              done
              echo "FATAL: session store unavailable, exiting"; exit 1
EOF
```

### 2.10 Check the scenarios
```bash
kubectl get pods -A | grep sre-
```
After about a minute you should see:

| Namespace | Expected status |
|---|---|
| sre-healthy | all `Running` and Ready |
| sre-imagepull | `ImagePullBackOff` |
| sre-crashloop | `CrashLoopBackOff` |
| sre-oom | `OOMKilled` / `CrashLoopBackOff` |
| sre-pending | 2 pods `Pending` |
| sre-service | `Running` (the problem is in the Service, not the pod) |
| sre-probes | `api` 0/1 `Running`, `gateway` restarting |
| sre-config | `CreateContainerConfigError` and `ContainerCreating` |
| sre-dependency | `checkout` crashing, `redis` Running |

---

## 3. Install HolmesGPT

```bash
helm repo add robusta https://robusta-charts.storage.googleapis.com && helm repo update
```
```bash
kubectl create namespace holmes
```

Pick **one** of the two AI options.

### Option A: OpenAI-compatible endpoint (LM Studio, vLLM, OpenAI, …)

**A.1 Store the URL and key in a Secret** (macOS shown, use the bash form on Linux):
```bash
read -rs "LLM_KEY?Endpoint API key: "; echo
```
```bash
kubectl -n holmes create secret generic holmes-llm --from-literal=OPENAI_API_KEY="$LLM_KEY" --from-literal=OPENAI_API_BASE="http://192.168.1.245:1234/v1"
```
```bash
unset LLM_KEY
```
Replace the URL with your endpoint, including `/v1`. If the model runs on your laptop, use your laptop's LAN IP, not `localhost`. Inside a pod, `localhost` means the pod itself.

**A.2 Create `holmes-values.yaml`:**
```yaml
additionalEnvVars:
  - name: OPENAI_API_KEY
    valueFrom: { secretKeyRef: { name: holmes-llm, key: OPENAI_API_KEY } }
  - name: OPENAI_API_BASE
    valueFrom: { secretKeyRef: { name: holmes-llm, key: OPENAI_API_BASE } }
  - name: MODEL
    value: gateway
modelList:
  gateway:
    api_key: "{{ env.OPENAI_API_KEY }}"
    api_base: "{{ env.OPENAI_API_BASE }}"
    model: openai/qwen/qwen3.5-9b
    temperature: 0
```
- `gateway` is a name you choose. Requests ask for the model by this name (`"model": "gateway"`).
- `openai/qwen/qwen3.5-9b` has two parts. `openai/` tells LiteLLM, the library inside Holmes, to use the OpenAI API format, and LiteLLM strips it before sending. Your endpoint receives `qwen/qwen3.5-9b`, the model's real ID. **Keep the `openai/` prefix**, or you get `LLM Provider NOT provided`.
- For a local model, set the **context length to at least 32K** (in LM Studio: the model's load settings). Holmes sends 20K to 30K tokens per step, and an 8K default fails with `exceeds the available context size`.

### Option B: Gemini

**B.1 Store the key in a Secret:**
```bash
read -rs "GEMINI_API_KEY?Gemini API key: "; echo
```
Check the key. This should print `models/gemini-3.1-flash-lite`:
```bash
curl -s https://generativelanguage.googleapis.com/v1beta/models -H "x-goog-api-key: $GEMINI_API_KEY" | jq -r '.models[].name' | grep flash-lite
```
```bash
kubectl -n holmes create secret generic gemini-llm --from-literal=GEMINI_API_KEY="$GEMINI_API_KEY"
```
```bash
unset GEMINI_API_KEY
```

**B.2 Create `holmes-values.yaml`:**
```yaml
additionalEnvVars:
  - name: GEMINI_API_KEY
    valueFrom: { secretKeyRef: { name: gemini-llm, key: GEMINI_API_KEY } }
  - name: TOOL_SCHEMA_NO_PARAM_OBJECT_IF_NO_PARAMS   # required by Holmes for Gemini
    value: "true"
  - name: MODEL
    value: gemini
modelList:
  gemini:
    api_key: "{{ env.GEMINI_API_KEY }}"
    model: gemini/gemini-3.1-flash-lite
    temperature: 1
```
- `gemini-3.1-flash-lite` is the cheapest stable Gemini model that supports tool calling: $0.25 per 1M input tokens and $1.50 per 1M output tokens ([pricing](https://ai.google.dev/gemini-api/docs/pricing), as of 2026-10-01). One Holmes question can use about 250K input tokens, roughly $0.07.
- On the **free tier**, Google may use your prompts to improve its products, and those prompts include pod names, logs and events. Fine for practice clusters. Think twice for real ones.
- If you see `503 This model is currently experiencing high demand`, that's Google's capacity, not your setup. Wait a minute and try again.
- Requests ask for this model as `"model": "gemini"`.

### 3.3 Install the chart (both options)
```bash
helm install holmes robusta/holmes --version 0.42.0 -n holmes -f holmes-values.yaml --wait --timeout 8m
```
```bash
kubectl -n holmes get pods
```
Wait until `holmes-holmes-…` shows `1/1 Running`.

### 3.4 Test Holmes before adding Slack
Terminal 1 (leave it running):
```bash
kubectl -n holmes port-forward svc/holmes-holmes 5050:80
```
Terminal 2 (use `"model":"gemini"` if you chose Option B):
```bash
curl -sS --max-time 600 -X POST http://localhost:5050/api/chat -H 'Content-Type: application/json' -d '{"ask":"why does the shop service in namespace sre-service have no endpoints?","model":"gateway"}' | jq -r '.analysis // .detail'
```
You should get the root cause (the `shopp` typo) after 20 seconds to a few minutes. If this fails, fix it first using **Troubleshooting**, because Slack won't work either.

Watch Holmes work live (optional, Terminal 3):
```bash
kubectl -n holmes logs deploy/holmes-holmes -f --tail=0
```

---

## 4. Create the Slack app

1. Go to [api.slack.com/apps](https://api.slack.com/apps), click **Create New App**, choose **From a manifest**, and pick your workspace.
2. Paste this on the **YAML** tab, then click **Create**:
```yaml
display_information:
  name: Holmes
  description: Asks HolmesGPT about the Kubernetes cluster
features:
  app_home:
    messages_tab_enabled: true
    messages_tab_read_only_enabled: false
  bot_user:
    display_name: holmes
    always_online: true
oauth_config:
  scopes:
    bot: [app_mentions:read, chat:write, im:history, im:read, im:write, reactions:write]
settings:
  event_subscriptions:
    bot_events: [app_mention, message.im]
  interactivity:
    is_enabled: true
  org_deploy_enabled: false
  socket_mode_enabled: true
  token_rotation_enabled: false
```
3. **App token:** go to **Basic Information**, then **App-Level Tokens**, then **Generate Token and Scopes**. Name it `socket`, add the scope `connections:write`, and click **Generate**. Copy the token that starts with **`xapp-`**.
4. **Bot token:** go to **Install App**, then **Install to Workspace**, then **Allow**. Copy the **Bot User OAuth Token** that starts with **`xoxb-`**.
5. **Interactivity** (needed for the buttons): in the left sidebar, under **Features**, open **Interactivity & Shortcuts**. Make sure it's **On**, then click **Save Changes**. With Socket Mode there's no Request URL to fill in.
6. **Your member ID** (only you will be allowed to approve fixes): in Slack, click your profile picture, then **Profile**, then **⋮**, then **Copy member ID**. It looks like `U0C26HTL0HW`.
7. In the channel where you want Holmes, type `/invite @holmes`. Direct messages work without this.

| Token | Starts with | Used for |
|---|---|---|
| App token | `xapp-` | Keeping the Socket Mode connection open, so Slack can send the bot your messages and button clicks |
| Bot token | `xoxb-` | Posting replies and adding reactions |

---

## 5. Store the Slack tokens

```bash
read -rs "SLACK_BOT_TOKEN?Bot token (xoxb-...): "; echo
```
```bash
read -rs "SLACK_APP_TOKEN?App token (xapp-...): "; echo
```
```bash
kubectl -n holmes create secret generic holmes-slack --from-literal=SLACK_BOT_TOKEN="$SLACK_BOT_TOKEN" --from-literal=SLACK_APP_TOKEN="$SLACK_APP_TOKEN"
```
```bash
unset SLACK_BOT_TOKEN SLACK_APP_TOKEN
```

---

## 6. Deploy the bot

### 6.1 Save the bot code as `bot.py`

```python
"""Slack bot that forwards @mentions and DMs to the in-cluster HolmesGPT API and replies in a thread.

When Holmes wants to run a command that changes the cluster, it pauses and the bot posts
Approve / Reject buttons. Only SLACK_APPROVER_ID can press them; the decision is sent back
to Holmes, which runs (or skips) the command and continues the investigation.
"""
import json
import logging
import os
import re
import threading
import uuid

import requests
from slack_bolt import App
from slack_bolt.adapter.socket_mode import SocketModeHandler

logging.basicConfig(level=logging.INFO, format="%(asctime)s %(levelname)s %(message)s")
log = logging.getLogger("holmes-slackbot")

HOLMES_URL = os.environ.get("HOLMES_URL", "http://holmes-holmes.holmes.svc.cluster.local/api/chat")
HOLMES_MODEL = os.environ.get("HOLMES_MODEL", "gateway")
APPROVER_ID = os.environ.get("SLACK_APPROVER_ID", "").strip()
SLACK_CHUNK = 3500  # Slack truncates very long messages, so long answers are split

app = App(token=os.environ["SLACK_BOT_TOKEN"])

# Investigations paused on an approval, keyed by a random id. Lost if the bot restarts.
PENDING = {}
PENDING_LOCK = threading.Lock()


def to_slack_markdown(text):
    # Holmes answers in standard Markdown; Slack uses *bold* and has no headings.
    text = re.sub(r"\*\*(.+?)\*\*", r"*\1*", text or "")
    return re.sub(r"^#{1,6}\s*(.+)$", r"*\1*", text, flags=re.MULTILINE)


def reply(client, channel, thread_ts, text):
    for i in range(0, len(text), SLACK_CHUNK):
        client.chat_postMessage(channel=channel, thread_ts=thread_ts, text=text[i:i + SLACK_CHUNK])


def call_holmes(payload):
    """POST to Holmes in streaming mode (the only mode that supports approvals).

    Returns (final_event, data, tools_run) where final_event is ai_answer_end, approval_required or error.
    """
    body = {"model": HOLMES_MODEL, "stream": True, "enable_tool_approval": True, **payload}
    tools_run = 0
    event = None
    with requests.post(HOLMES_URL, json=body, stream=True, timeout=(10, 900)) as resp:
        for line in resp.iter_lines(decode_unicode=True):
            if line.startswith("event: "):
                event = line[len("event: "):]
            elif line.startswith("data: "):
                data = json.loads(line[len("data: "):])
                if event == "start_tool_calling":
                    tools_run += 1
                elif event in ("ai_answer_end", "approval_required", "error"):
                    return event, data, tools_run
    return "error", {"description": f"Holmes closed the stream without an answer (HTTP {resp.status_code})"}, tools_run


def describe_command(approval):
    params = approval.get("params") or {}
    return params.get("command") or approval.get("description") or json.dumps(params)


def process(client, channel, thread_ts, payload):
    """Run one Holmes round and post either the answer or approval buttons."""
    try:
        event, data, tools_run = call_holmes(payload)
    except requests.RequestException as e:
        reply(client, channel, thread_ts, f":x: Could not reach Holmes at {HOLMES_URL}: {e}")
        return

    if event == "error":
        reply(client, channel, thread_ts, f":x: Holmes returned an error:\n```{data.get('description') or data.get('msg') or data}```")
        return

    if event == "ai_answer_end":
        costs = (data.get("metadata") or {}).get("costs") or {}
        footer = f"\n\n_{tools_run} tools run · {costs.get('prompt_tokens', '?')} prompt tokens · model {HOLMES_MODEL}_"
        reply(client, channel, thread_ts, to_slack_markdown(data.get("analysis")) + footer)
        return

    # approval_required: park the conversation and ask the approver
    key = uuid.uuid4().hex[:12]
    approvals = data.get("pending_approvals") or []
    with PENDING_LOCK:
        PENDING[key] = {
            "history": data.get("conversation_history"),
            "commands": {a["tool_call_id"]: describe_command(a) for a in approvals},
            "decisions": {},
            "channel": channel,
            "thread_ts": thread_ts,
        }
    if data.get("analysis"):
        reply(client, channel, thread_ts, to_slack_markdown(data["analysis"]))
    who = f"<@{APPROVER_ID}>" if APPROVER_ID else "the approver (SLACK_APPROVER_ID is not set, so nobody can approve)"
    for a in approvals:
        value = f"{key}|{a['tool_call_id']}"
        client.chat_postMessage(
            channel=channel,
            thread_ts=thread_ts,
            text=f"Holmes wants to run: {describe_command(a)}",
            blocks=[
                {"type": "section", "text": {"type": "mrkdwn", "text": f":warning: *Holmes wants to change the cluster* ({tools_run} tools run so far). Waiting for {who}:\n```{describe_command(a)}```"}},
                {"type": "actions", "elements": [
                    {"type": "button", "action_id": "holmes_approve", "style": "primary", "text": {"type": "plain_text", "text": "Approve"}, "value": value},
                    {"type": "button", "action_id": "holmes_reject", "style": "danger", "text": {"type": "plain_text", "text": "Reject"}, "value": value},
                ]},
            ],
        )


def ask(question, channel, message_ts, thread_ts, client):
    if not question:
        reply(client, channel, thread_ts, "Ask me about the cluster, e.g. `@holmes why is report-worker in namespace sre-oom restarting?`")
        return
    log.info("question in %s: %s", channel, question)
    client.reactions_add(channel=channel, timestamp=message_ts, name="mag")
    reply(client, channel, thread_ts, ":mag: Investigating, this can take a few minutes...")
    process(client, channel, thread_ts, {"ask": question})


def decide(body, client, approved):
    user = body["user"]["id"]
    channel = body["channel"]["id"]
    action = body["actions"][0]
    key, tool_call_id = action["value"].split("|", 1)

    if not APPROVER_ID or user != APPROVER_ID:
        client.chat_postEphemeral(channel=channel, user=user, text="Only the configured approver can approve or reject Holmes' changes.")
        return

    with PENDING_LOCK:
        job = PENDING.get(key)
        if job is None or tool_call_id in job["decisions"]:
            client.chat_postEphemeral(channel=channel, user=user, text="This request is no longer waiting (already decided, or the bot restarted). Ask Holmes again.")
            return
        decision = {"tool_call_id": tool_call_id, "approved": approved}
        if not approved:
            decision["feedback"] = "Rejected by the user in Slack. Do not run this command; suggest an alternative instead."
        job["decisions"][tool_call_id] = decision
        all_decided = len(job["decisions"]) == len(job["commands"])
        if all_decided:
            PENDING.pop(key)

    command = job["commands"][tool_call_id]
    verdict = ":white_check_mark: Approved" if approved else ":no_entry: Rejected"
    client.chat_update(
        channel=channel,
        ts=body["message"]["ts"],
        text=f"{verdict} by <@{user}>",
        blocks=[{"type": "section", "text": {"type": "mrkdwn", "text": f"{verdict} by <@{user}>:\n```{command}```"}}],
    )
    log.info("%s %s by %s", "approved" if approved else "rejected", tool_call_id, user)

    if all_decided:
        reply(client, job["channel"], job["thread_ts"], ":gear: Sending your decision to Holmes...")
        process(client, job["channel"], job["thread_ts"], {
            "ask": "",
            "conversation_history": job["history"],
            "tool_decisions": list(job["decisions"].values()),
        })


@app.action("holmes_approve")
def on_approve(ack, body, client):
    ack()
    decide(body, client, approved=True)


@app.action("holmes_reject")
def on_reject(ack, body, client):
    ack()
    decide(body, client, approved=False)


@app.event("app_mention")
def on_mention(event, client):
    question = re.sub(r"<@[^>]+>", "", event["text"]).strip()
    ask(question, event["channel"], event["ts"], event.get("thread_ts") or event["ts"], client)


@app.event("message")
def on_direct_message(event, client):
    if event.get("channel_type") != "im" or event.get("bot_id") or event.get("subtype"):
        return
    ask(event.get("text", "").strip(), event["channel"], event["ts"], event.get("thread_ts") or event["ts"], client)


if __name__ == "__main__":
    log.info("connecting to Slack, forwarding to %s (model %s), approver %s", HOLMES_URL, HOLMES_MODEL, APPROVER_ID or "NOT SET")
    SocketModeHandler(app, os.environ["SLACK_APP_TOKEN"]).start()
```

### 6.2 Load the code into the cluster
```bash
kubectl -n holmes create configmap holmes-slackbot --from-file=bot.py
```

### 6.3 Save the bot settings as `bot-deployment.yaml`
Set `HOLMES_MODEL` to your model name from step 3 (`gateway` or `gemini`) and `SLACK_APPROVER_ID` to your member ID.
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: holmes-slackbot
  namespace: holmes
spec:
  replicas: 1
  selector:
    matchLabels: { app: holmes-slackbot }
  template:
    metadata:
      labels: { app: holmes-slackbot }
    spec:
      containers:
        - name: bot
          image: python:3.12-slim
          # Installs the Slack SDK on start, so no custom image has to be built
          command: ["sh", "-c", "pip install --no-cache-dir -q slack_bolt requests && python /app/bot.py"]
          env:
            - { name: PYTHONUNBUFFERED, value: "1" }
            - { name: HOLMES_MODEL, value: gateway }
            - { name: SLACK_APPROVER_ID, value: "U0C26HTL0HW" }
            - name: SLACK_BOT_TOKEN
              valueFrom: { secretKeyRef: { name: holmes-slack, key: SLACK_BOT_TOKEN } }
            - name: SLACK_APP_TOKEN
              valueFrom: { secretKeyRef: { name: holmes-slack, key: SLACK_APP_TOKEN } }
          volumeMounts:
            - { name: code, mountPath: /app }
          resources:
            requests: { cpu: 50m, memory: 128Mi }
            limits: { memory: 256Mi }
      volumes:
        - name: code
          configMap: { name: holmes-slackbot }
```

### 6.4 Start it and check that it connected
```bash
kubectl apply -f bot-deployment.yaml
```
```bash
kubectl -n holmes logs deploy/holmes-slackbot -f
```
The first start takes about 30 seconds while it installs the Slack library. You want to see:
```
connecting to Slack, forwarding to http://holmes-holmes... (model gateway), approver U0C26HTL0HW
Starting to receive messages from a new connection
```
At this point you can already ask Holmes questions in Slack. The next step is only needed if you want it to apply fixes.

---

## 7. (Optional) Allow Holmes to apply approved fixes

Out of the box, Holmes is **read-only**. When you click Approve, it runs the command as its own Kubernetes account, and Kubernetes would refuse with `Forbidden`. This gives that account a few write permissions, **only in the namespaces you list**:

```bash
for ns in sre-healthy sre-imagepull sre-crashloop sre-oom sre-pending sre-service sre-probes sre-config sre-dependency; do
kubectl apply -f - <<EOF
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata: { name: holmes-remediate, namespace: $ns }
rules:
  - { apiGroups: [apps], resources: [deployments, statefulsets], verbs: [patch, update] }
  - { apiGroups: [""], resources: [services], verbs: [patch, update] }
  - { apiGroups: [""], resources: [pods], verbs: [delete] }
  - { apiGroups: [""], resources: [secrets, configmaps], verbs: [create] }
  - { apiGroups: [""], resources: [persistentvolumeclaims], verbs: [create, delete] }
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata: { name: holmes-remediate, namespace: $ns }
roleRef: { apiGroup: rbac.authorization.k8s.io, kind: Role, name: holmes-remediate }
subjects:
  - { kind: ServiceAccount, name: holmes-holmes-service-account, namespace: holmes }
EOF
done
```
Check it. The first should print `yes`, the second `no`:
```bash
kubectl auth can-i patch services -n sre-service --as=system:serviceaccount:holmes:holmes-holmes-service-account
```
```bash
kubectl auth can-i patch services -n kube-system --as=system:serviceaccount:holmes:holmes-holmes-service-account
```

What this does and doesn't allow:
- It can **create** Secrets but not **read** them, so it can't leak passwords.
- It can **patch** Deployments but not **delete** them, so the worst case is an edit you can revert.
- It can't touch `holmes`, `kube-system`, nodes or permissions, so it can't change itself or give itself more power.
- Nothing runs without your **Approve** click.

---

## 8. Try it

DM **Holmes** in Slack, or mention it in a channel you invited it to:
```
@holmes why is the checkout pod in namespace sre-dependency failing?
```
Holmes adds 🔍, replies "Investigating…", and a few minutes later posts the answer in the thread with ✅.

**Test an approved fix** on the selector typo:
```
the shop service in sre-service has no endpoints. find the cause and fix it
```
1. Holmes posts the cause, then: ⚠️ *Holmes wants to change the cluster*:
   `kubectl patch service shop -n sre-service -p '{"spec": {"selector": {"app": "shop"}}}'`, with **Approve** and **Reject**.
2. Click **Approve**. The message changes to "✅ Approved by @you", and Holmes runs the command, checks the result and reports back.
3. Confirm it yourself. Under ENDPOINTS you should now see a pod IP:
```bash
kubectl get endpoints -n sre-service shop
```
4. Break it again for the next demo:
```bash
kubectl patch svc shop -n sre-service -p '{"spec":{"selector":{"app":"shopp"}}}'
```

**Read the command before you click Approve.** The AI's fix isn't always right. For `sre-oom`, models often suggest raising the memory limit, which doesn't help because `tail /dev/zero` keeps using more. Click **Reject**, and Holmes writes the fix up as a suggestion instead.

---

## 9. How it works

1. **Question.** Slack sends your message to the bot over the Socket Mode connection the bot opened. The bot removes the `@holmes` mention and sends the question to Holmes inside the cluster (`http://holmes-holmes.holmes.svc.cluster.local/api/chat`) with `stream: true` and `enable_tool_approval: true`. Approvals only work in streaming mode.
2. **Investigation.** Holmes sends the question to the AI model together with a list of tools. The model asks for tools (`kubectl get`, `describe`, `logs`, `events`…), Holmes runs them in its pod and sends back the results, and this repeats until the model has an answer.
3. **The gate.** Holmes' built-in `bash` tool sorts every command. Read-only commands run immediately, `sudo` and `su` are blocked, and anything that changes the cluster (`kubectl patch`, `delete`, …) **needs approval**. When that happens, Holmes stops and returns an `approval_required` event with the exact command, its ID, and the conversation so far.
4. **Buttons.** The bot stores that conversation and posts the command with Approve and Reject (Slack Block Kit buttons). Slack sends your click down the same Socket Mode connection. The bot checks the click came from `SLACK_APPROVER_ID`, then sends Holmes the stored conversation plus your decision (`tool_decisions`). Holmes picks up exactly where it stopped.
5. **Answer.** Holmes finishes with an `ai_answer_end` event, and the bot converts the Markdown to Slack's format and posts it in the thread.

The paused conversations live in the bot's memory. If the bot restarts, older buttons stop working, so ask again.

---

## 10. Everyday commands

| Task | Command |
|---|---|
| Bot and Holmes status | `kubectl -n holmes get pods` |
| Bot logs (connections, questions, approvals) | `kubectl -n holmes logs deploy/holmes-slackbot -f` |
| Holmes logs (tool calls, model errors) | `kubectl -n holmes logs deploy/holmes-holmes -f --tail=0` |
| Reload after editing `bot.py` | `kubectl -n holmes create configmap holmes-slackbot --from-file=bot.py --dry-run=client -o yaml \| kubectl apply -f - && kubectl -n holmes rollout restart deploy/holmes-slackbot` |
| Change the approver | `kubectl -n holmes set env deploy/holmes-slackbot SLACK_APPROVER_ID=U...` |
| Change the AI model | edit `holmes-values.yaml`, then `helm upgrade holmes robusta/holmes --version 0.42.0 -n holmes -f holmes-values.yaml` |
| Rebuild all scenarios | rerun the step 2 commands (`kubectl apply` resets the broken settings) |

---

## 11. Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| Bot pod shows `CreateContainerConfigError` | The `holmes-slack` Secret doesn't exist yet | Do step 5; the pod starts on its own |
| Bot log shows `invalid_auth` | Tokens swapped or wrong | `kubectl -n holmes delete secret holmes-slack`, redo step 5, restart the bot |
| No reply in a channel | Bot not invited | `/invite @holmes` in that channel |
| Clicking a button does nothing | Interactivity is off | Slack app, **Interactivity & Shortcuts**, **On**, **Save** |
| "Only the configured approver…" | `SLACK_APPROVER_ID` isn't your ID | `kubectl -n holmes set env deploy/holmes-slackbot SLACK_APPROVER_ID=U...` |
| Approved, then `Forbidden` | Holmes lacks write permission there | Step 7, for that namespace |
| `LLM Provider NOT provided` | `openai/` prefix missing from `model:` | Use `openai/<model-id>` and `helm upgrade` |
| `exceeds the available context size (8192 tokens)` | Local model loaded with a small context | Raise the context length to 32K or more in LM Studio and reload the model |
| `503 … high demand` (Gemini) | Google is at capacity | Wait a minute and retry, or try another Gemini model |
| `Connection refused` to your endpoint | URL uses `localhost` | Use the machine's LAN IP in the `holmes-llm` Secret |
| Holmes very slow | Another tool (for example K8sGPT) is using the same local model | Pause the other tool, or give it a smaller model |
| `curl: Empty reply from server` | The port-forward attached to an old pod during a restart | Stop and restart the port-forward |
| Answer says "API server out of memory" | kubectl crashed inside the Holmes pod; the model misread it | Ignore that part; restarting Holmes usually clears it |

---

## 12. Clean up

Remove only the Slack bot:
```bash
kubectl -n holmes delete deploy/holmes-slackbot configmap/holmes-slackbot secret/holmes-slack
```
Remove Holmes' fix permissions:
```bash
for ns in sre-healthy sre-imagepull sre-crashloop sre-oom sre-pending sre-service sre-probes sre-config sre-dependency; do kubectl -n $ns delete role,rolebinding holmes-remediate; done
```
Remove Holmes:
```bash
helm uninstall holmes -n holmes
```
Remove the scenarios:
```bash
kubectl delete ns sre-healthy sre-imagepull sre-crashloop sre-oom sre-pending sre-service sre-probes sre-config sre-dependency
```
Remove everything, cluster included:
```bash
kind delete cluster --name sre
```
Finally, delete the **Holmes** app at [api.slack.com/apps](https://api.slack.com/apps). That also invalidates both Slack tokens.
