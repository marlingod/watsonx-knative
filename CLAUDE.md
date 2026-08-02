# watsonx-knative — working notes (non-obvious gotchas only)

Keep this LEAN. For each line ask "would removing this cause a mistake?" If not, cut it.

A scale-to-zero **Knative Serving** Service that proxies HTTP inference requests to **IBM watsonx.ai**.

## Commands the agent can't guess
> Finalize after the design pass picks the runtime (see `docs/design/`). Placeholders:
- Build image: `docker build -t <registry>/watsonx-proxy:<tag> .`
- Run tests: `<TBD after runtime chosen>`   (note any counter-intuitive settings/env here)
- Lint/typecheck: `<TBD>`
- Local run: `docker run -p 8080:8080 --env-file .env.local <image>`
- Deploy: see the `watsonx-knative-controlled-deploy` skill (`kn` / `kubectl apply`).

## Conventions that DIFFER from defaults
- Branch off `main`; never commit secrets.
- The container **must listen on `$PORT` (default 8080)** — Knative injects `PORT`.

## Architecture
- Stack: Knative Serving + IBM watsonx.ai. The service is a thin, stateless proxy: receive request →
  (cached IAM bearer token) → call watsonx `POST /ml/v1/text/generation?version=…` → return. See
  `docs/design/knative-watsonx-inference-design.md` for the design decision + rationale.

## Non-obvious gotchas
- **IAM token is not the API key.** Exchange the IBM Cloud API key for an IAM **bearer token** (TTL ~1h)
  and **cache + refresh it in memory** — never mint one per request (adds latency, hits rate limits).
- **`version` query param is mandatory** on watsonx calls, and `project_id` (or `space_id`) is required
  in the body.
- **Cold start**: on scale-from-zero the first request must also mint a fresh IAM token. Budget for it,
  or pin `min-scale=1` for latency-sensitive paths.
- One `containerPort` only; revisions are immutable (a new config = a new revision).

## Verification rule
- Nothing is "done" without a machine-checkable pass/fail: image builds, `kn service describe` shows
  **`Ready=True`**, and a **live `curl`** returns a real inference response. Never ship on "looks right".

## Danger zones (human-only / extra care)
- `kubectl apply` / `kn service create`, k8s **Secrets** (the watsonx API key), image push to the registry.
