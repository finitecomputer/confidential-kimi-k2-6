# confidential-kimi-k2-6

Current `tinfoil-config.yml` routes the public shim to the Finite Private
limiter on port `8002`; the limiter forwards admitted requests to vLLM on
`8001` using the internal vLLM API key.

The config exposes:

- `/live` for limiter process liveness.
- `/health` and `/ready` for deep readiness. The hardened limiter checks both
  vLLM and the Finite Core usage API before returning `200`.

The timeout/readiness config ships with the digest-pinned hardened limiter
image built from `/Users/futurepaul/dev/finite/finitecomputer`.

Finite Private rollout uses this published digest-pinned limiter image:

```text
ghcr.io/finitecomputer/finite-private-limiter:2026-06-18.deep-health.1@sha256:32d357c5d01bfa027381c07b357a4ed96602dabede5656b18907c53beaf16d18
```

Package visibility/access is confirmed: anonymous `docker buildx imagetools
inspect` succeeds against the pinned image.

The measured release is
`https://github.com/finitecomputer/confidential-kimi-k2-6/releases/tag/v2026-06-18-deep-health-limiter`
with deployment digest:

```text
0a21a95a9042632d7dbd11b83721f04caa823e009f0b834f99029a14b8c3becd
```

Required Tinfoil secrets:

- `FINITE_USAGE_API_SERVICE_KEY`
- `VLLM_INTERNAL_API_KEY`
- `VLLM_API_KEY`

Set `VLLM_API_KEY` and `VLLM_INTERNAL_API_KEY` to the same value for the first
limiter rollout.

Live rollout on 2026-06-18:

- Tinfoil container `kimi-k2-6` is running tag
  `v2026-06-18-deep-health-limiter`.
- First relaunch attempt at `2026-06-18T03:48:46Z` failed because the tag had a
  plain GitHub Release without measured Tinfoil assets. The measured release was
  published at `2026-06-18T03:50:44Z`.
- Relaunch was accepted at `2026-06-18T03:51:04Z`; Tinfoil reported `ready` and
  public `/health` returned deep limiter readiness at `2026-06-18T04:25:28Z`,
  for 2,064 seconds from accepted relaunch to ready.
- Public `/health` returned the hardened limiter readiness payload with both
  `upstream` and `usageApi` healthy.
- A negative public chat canary with an intentionally invalid Finite Private key
  returned `401 invalid_api_key`, proving shim -> limiter -> Core admission.
  A positive reserve/proxy/settle canary still requires a real canary API key.

Previous live rollout on 2026-05-26:

- Tinfoil container `kimi-k2-6` was running tag
  `v2026-05-26-private-limiter`.
- Relaunch started at `2026-05-26T22:55:58Z`; Tinfoil reported `ready` at
  `2026-05-26T23:30:56Z`, for 2,098 seconds of downtime.
- Public raw HTTPS canaries passed for `/health`, non-streaming
  `/v1/chat/completions`, streaming `/v1/chat/completions`, Core settlement,
  and revoked-key denial.
- `tinfoil http` verified requests currently fail with `tcbInfo has expired`;
  raw HTTPS through the public shim is working.

Keep a rollback commit that restores `shim.upstream-port: 8001` for emergency
direct-vLLM recovery.
