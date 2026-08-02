---
name: watsonx-knative-controlled-deploy
description: Deploy the watsonx-knative inference proxy to the cluster safely (build → push → apply → wait Ready → live curl), with rollback. Use when deploying or when the service is unhealthy.
---

# Controlled Deploy — watsonx-knative

A stateless Knative Service. Revisions are immutable; traffic shifts between them, so rollback is a
traffic split, not a restore. **Verify every step; the watsonx API key lives only in a k8s Secret.**

## Prerequisites (once)
- The API key Secret exists (never in the image/manifest):
  `kubectl create secret generic watsonx-creds --from-literal=WATSONX_APIKEY=<key>`
- Region/project are config: `WATSONX_URL`, `WATSONX_PROJECT_ID` (env or ConfigMap).

## Procedure
1. **Build + push** the image (immutable tag, e.g. git SHA):
   `docker build -t $REG/watsonx-proxy:$SHA . && docker push $REG/watsonx-proxy:$SHA`
2. **Apply** the Service (points at the new tag → creates a new revision):
   `kubectl apply -f service.yaml`   (or `kn service update watsonx-proxy --image $REG/watsonx-proxy:$SHA`)
3. **Wait for Ready**: `kubectl wait ksvc/watsonx-proxy --for=condition=Ready --timeout=180s`
   (or `kn service describe watsonx-proxy` → `Ready=True`).
4. **Verify end-to-end** — a live inference, not just health:
   `URL=$(kn service describe watsonx-proxy -o url); curl -sS -X POST "$URL/infer" -d '{"input":"ping"}' -H 'content-type: application/json'`
   Expect a real watsonx response. (First call after scale-from-zero is slow — that's the cold start.)

## Rollback (traffic, not restore)
- `kn revision list` → find the last-good revision.
- `kn service update watsonx-proxy --traffic <good-revision>=100` (instant; no rebuild).

## ⚠️ Gotchas (fill in as you hit them)
- Cold-start latency on scale-from-zero includes a fresh IAM token mint — set a startup probe /
  `min-scale=1` if the SLO needs it.
- `imagePullSecrets` needed for a private registry.
- `timeoutSeconds` on the Service must exceed watsonx worst-case latency or requests get cut off.
- <record each real incident + fix here — this section is the payoff>

## Verdict format
```
Deployed watsonx-proxy @ <sha>. Revision: <name>  Ready=True
E2E: POST /infer -> <status> (real inference: yes/no)
```
