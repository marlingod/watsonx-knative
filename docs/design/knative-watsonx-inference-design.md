# Design — Scale-to-Zero Knative Service serving IBM watsonx.ai inference

**Status:** Proposed · **Produced by:** graph-engineering blueprint (3 parallel research workers →
verify → merge). Verified claims are marked ✅ (live-confirmed) vs ⚠️ (documented/known, confirm for
your account/version).

## Decision summary
Build a **thin, stateless HTTP proxy** deployed as a Knative Serving `Service` that: receives a request →
uses a **cached IAM bearer token** → calls watsonx.ai `POST /ml/v1/text/generation` → returns the result.

- **Runtime:** **Node/Fastify calling the watsonx REST API directly** (recommended, because `min-scale=0`
  is a cost driver and the Python `ibm-watsonx-ai` SDK is the single biggest cold-start tax — 200-500ms
  of import time, larger image). **Python/FastAPI is an acceptable fallback** if the team is Python-only,
  but then treat SDK-import-time reduction (or vendoring just the auth+call logic) as required work.
- **Autoscaler:** **KPA** (only class that scales to zero; understands request concurrency, the right
  signal for an I/O-bound proxy). ✅
- **Scaling default:** `min-scale=0` with a tuned pod-retention; move to `min-scale=1` only if a latency
  SLO proves the cold-start tail unacceptable.
- **Non-optional v1 scope:** IAM token cache + proactive refresh, retry-with-backoff + circuit breaker,
  and refresh-token-once-on-401. A happy-path-only proxy will fail exactly under this architecture's
  normal conditions (bursty scale-from-zero + transient upstream blips).

## Architecture
```
client ──HTTP──▶ Knative Service (KPA, scale-to-zero)
                    └─ proxy container (:8080, $PORT)
                         • cached IAM bearer token (mint at startup, refresh <5min to expiry)
                         • POST https://<region>.ml.cloud.ibm.com/ml/v1/text/generation?version=YYYY-MM-DD
                         • retry/backoff + circuit breaker + 401→remint→retry-once
                    ▲
                    └─ watsonx API key from a k8s Secret; region/project_id/version from a ConfigMap
```

## Knative Service (concrete)
Autoscaling annotations go under `spec.template.metadata.annotations` (NOT top-level) ✅.
`containerConcurrency`, `timeoutSeconds`, `imagePullSecrets` are Revision `spec` fields, not annotations ✅.
```yaml
apiVersion: serving.knative.dev/v1
kind: Service
metadata: { name: watsonx-proxy }
spec:
  template:
    metadata:
      annotations:
        autoscaling.knative.dev/class: "kpa.autoscaling.knative.dev"   # KPA (scale-to-zero) ✅
        autoscaling.knative.dev/min-scale: "0"                          # 0 = scale to zero
        autoscaling.knative.dev/max-scale: "10"
        autoscaling.knative.dev/target: "50"                            # soft concurrency target → drives scaling
        autoscaling.knative.dev/scale-to-zero-pod-retention-period: "2m" # absorb bursty-but-not-idle traffic
    spec:
      containerConcurrency: 100    # MERGE NOTE: not 0/unbounded. Soft target (above) drives scaling;
                                   # this generous hard cap protects ONE pod from unbounded in-flight
                                   # (FD/memory exhaustion) during scale-up. Pair with a bounded HTTP
                                   # client connection pool in the app.
      timeoutSeconds: 300          # ⚠️ set deliberately > your outbound client timeout + IAM mint;
                                   # for streaming use responseStartTimeoutSeconds/idleTimeoutSeconds
      containers:
        - image: <registry>/watsonx-proxy:<sha>
          ports: [{ containerPort: 8080 }]      # exactly ONE port allowed ✅
          envFrom:
            - configMapRef: { name: watsonx-config }     # WATSONX_URL, WATSONX_PROJECT_ID, WATSONX_API_VERSION
          env:
            - name: WATSONX_APIKEY
              valueFrom: { secretKeyRef: { name: watsonx-credentials, key: WATSONX_APIKEY } }
          readinessProbe: { httpGet: { port: 8080, path: /healthz } }
          # NOTE: liveness is NOT rewritten by queue-proxy — mirror it as a readiness check too ✅
          resources:
            requests: { cpu: 100m, memory: 256Mi }
            limits:   { cpu: "1",  memory: 512Mi }
```
Scale-to-zero is a **global** ConfigMap feature (`enable-scale-to-zero`, default true; grace period 30s) —
per-service you only set `min-scale`/`max-scale`/retention ✅.

## watsonx auth + call (✅ live-probed against IBM prod)
**Mint the IAM token (cache it; do NOT do this per request):**
```
POST https://iam.cloud.ibm.com/identity/token
Content-Type: application/x-www-form-urlencoded
grant_type=urn:ibm:params:oauth:grant-type:apikey&apikey=<WATSONX_APIKEY>
→ { "access_token": "...", "expires_in": 3600, "expiration": <epoch> }   # TTL ~3600s ✅
```
- `refresh_token` is `"not_supported"` — re-run the same apikey→token POST before expiry ✅. Refresh when
  `<5min` remain; also remint + retry once on a `401`.
- The API key is **only** used to mint the token; the inference API rejects requests without a Bearer token ✅.

**Inference call:**
```
POST https://<region>.ml.cloud.ibm.com/ml/v1/text/generation?version=YYYY-MM-DD   # version MANDATORY ✅
Authorization: Bearer <access_token>
{ "model_id": "...", "input": "...", "parameters": {...}, "project_id": "<id>" }   # project_id XOR space_id ✅
```
- Pin `version` to a **tested constant** date (not "today") — malformed/missing → `400 invalid_version_date_pattern` ✅.
- `project_id`/`space_id`/model must live in the **same region** as the base URL ⚠️.

## Secrets vs config
| Item | Kind | Where |
|---|---|---|
| `WATSONX_APIKEY` | **Secret** | k8s Secret via `secretKeyRef` — never in image, manifest value, or logs |
| IAM bearer token | Secret, ephemeral | in-memory only; never persisted/logged |
| `WATSONX_URL` / `WATSONX_PROJECT_ID` / `WATSONX_API_VERSION` / `model_id` | Config | ConfigMap |

**Hardening:** prefer **External Secrets Operator** or **IBM Cloud Secrets Manager CSI** over a hand-made
Secret; enable etcd encryption-at-rest + RBAC on the Secret. If the cluster is IBM-managed (IKS/ROKS),
evaluate **Trusted Profiles / compute-resource IAM** to drop the static API key entirely ⚠️ (not available
on generic/other-cloud Knative).

## Security checklist (enforce)
- API key only in a Secret; never in image/manifest/logs (redact `Authorization` in middleware).
- IAM token in memory only; TLS to watsonx enforced (no cert-verify disable).
- Don't log prompt/completion bodies with PII by default (log metadata: id, latency, status, token counts).
- Least-privilege IAM (scoped to the one project/space). Cap incoming body size; bound outbound client pool.

## Failure modes → required handling
- `429` → exponential backoff + jitter, honor `Retry-After`; the IAM token endpoint also throttles (→ cache).
- `5xx` → bounded retries + **circuit breaker** (fail fast when open) so an outage doesn't hang every request.
- Token expiry mid-flight → invalidate cache, remint, **retry once** (not indefinitely).
- Cold-start timeout → test time-from-zero-to-first-response; set timeouts with margin (Knative `timeoutSeconds`
  > app client timeout).
- Region outage → circuit-break + distinguishable error; multi-region failover is a human decision (data residency).

## Cold-start vs cost
`min-scale=0` = cheapest but the first request after idle pays pod start + dep load + **IAM token mint** +
TLS. Node lean client ≈ 300-800ms ⚠️; naive Python+SDK ≈ 1-3s ⚠️. Mitigate: **pre-mint the IAM token at
pod startup** (readiness gates on it → out of the request path), tune `scale-to-zero-pod-retention-period`,
keep the image small. Use `min-scale=1` only if the SLO can't tolerate the tail; "hybrid off-hours" needs
an external scheduler (not native) — defer to a later optimization.

## Open questions for you (decide before build)
1. **Region** (`us-south`/`eu-de`/…) — latency, data residency, DR.
2. **Which watsonx model** — drives latency, cost/token, context window, and whether streaming is needed.
3. **Project vs deployment space** — prod usually targets a space (governance implications).
4. **Expected RPS + latency SLO** — decides `min-scale=0` viability and whether the Node/Python cold-start
   delta even matters.
5. **Streaming responses?** — changes endpoint (`/text/generation_stream`), timeouts, and error handling.

## Sources
Knative: knative.dev/docs/serving/{services, autoscaling/scale-to-zero, autoscaling/concurrency,
autoscaling/autoscaler-types, services/configure-probing, request-flow}; knative/specs API spec.
IBM: cloud.ibm.com/apidocs/watsonx-ai (+ live endpoint probes), cloud.ibm.com/docs/account IAM token.
