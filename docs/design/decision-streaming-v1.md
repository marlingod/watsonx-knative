# Decision — Streaming (SSE) in v1?

**Decision: DEFER to v2. Ship the non-streaming proxy first.** Produced by the graph-engineering
blueprint (FOR-node ∥ AGAINST-node → verify → merge).

## Why defer
1. **Streaming breaks v1's committed foundation.** The design report already makes
   retry+backoff + circuit-breaker + 401→re-mint→retry-once **non-optional v1 scope**. Those patterns
   assume you can safely resend a request — which is impossible once any SSE byte has been sent to the
   client. Adding streaming in v1 means either weakening that foundation or building both at once. Prove
   the resilient request/response loop first; layer streaming on a battle-tested base.
2. **YAGNI for internal users.** The UX win (token-by-token) is largest for human chat UIs. v1 is an
   internal proxy; unless a consumer is building an interactive UI, blocking responses are acceptable.
3. **Scale-to-zero economics.** A streaming request holds a `containerConcurrency` slot for the whole
   generation → longer-held connections, fewer concurrent users/pod, worse for a bursty scale-from-zero
   proxy.
4. **Hidden infra risk.** Ingress/queue-proxy layers may buffer SSE unless explicitly configured —
   "streaming" can silently degrade to one big flush, adding complexity for zero benefit.

## The FOR case is real — capture it for v2 (don't lose it)
The strongest pro-streaming insight is worth banking now: **streaming converts one hard
`responseStartTimeoutSeconds`/`timeoutSeconds` deadline into a renewable `idleTimeoutSeconds` budget.** A
proxy that emits an immediate SSE heartbeat (`: ping`) on connect — before IAM auth / the watsonx call —
satisfies `responseStartTimeoutSeconds` trivially and masks *connection* cold-start. **v2 streaming design
= heartbeat-before-first-token + deliberately tuned `responseStartTimeoutSeconds`/`idleTimeoutSeconds`**
(not Knative defaults). Note: this improves *perceived/operational robustness*, it does **not** reduce
actual cold-start or IAM-mint latency.

## Pull streaming into v1 ONLY IF (any):
- A concrete internal consumer is building an interactive/chat UI now (else you just push the same
  complexity onto their team).
- Typical completions routinely exceed ~10–15s (blocking becomes a real usability problem).
- Cluster is confirmed **Knative Serving ≥ 1.7** (the two timeout fields exist — verified: added in
  v1.7.0, ~Nov 2022, absent in v1.6.0) **and** the ingress is confirmed to pass SSE unbuffered.
- v2 won't ship within an acceptable window (so "defer" ≈ "never").

## Action items regardless of decision
- **Confirm the installed Knative Serving version** (pre-1.7 = the streaming timeout knobs don't exist).
- **30-min curl smoke test** of watsonx `/ml/v1/text/generation_stream` to lock the exact per-chunk SSE
  schema before any streaming work (both nodes flagged this as the one unverified gap).
- If/when streaming ships: wire **client-disconnect → cancel the upstream watsonx call** (else orphaned
  generations burn quota/cost) and **mid-stream errors as an `event: error` SSE frame** (status code is
  already 200 once bytes flow).

_Sources: IBM watsonx apidocs (generation_stream, live-probed); Knative Serving API ref +
`revision_types.go` at v1.6.0/v1.7.0 tags._
