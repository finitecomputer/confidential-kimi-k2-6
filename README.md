# confidential-kimi-k2-6

This repository name is historical. Current `tinfoil-config.yml` serves the
managed `glm-5-2` Finite Private model through the existing Tinfoil deployment.

The public shim routes to the Finite Private limiter on port `8002`; the
limiter reserves usage through Core before forwarding admitted requests to vLLM
on `8001` using the internal vLLM API key. Do not change
`shim.upstream-port: 8002` for the GLM rollout unless you are intentionally
bypassing Finite Private admission as an emergency rollback.

The GLM model-side config follows Tinfoil's `confidential-glm5-2` `v0.0.14`
release:

- `cvm-version: 0.10.4`
- model repo: `zai-org/GLM-5.2-FP8@a0b55e88465d1a06afece97bc8d6b366aff39089`
- image:
  `ghcr.io/tinfoilsh/confidential-glm5-2@sha256:8cc690cf5b1c26b0bc14894a7ca27890386b536930b69172678560220572648b`
- served model name: `glm-5-2`
- vLLM port: `8001`
- limiter port: `8002`

The config exposes:

- `/live` for limiter process liveness.
- `/health` and `/ready` for deep readiness. The hardened limiter checks both
  vLLM and the Finite Core usage API before returning `200`.

The timeout/readiness config still ships with the digest-pinned hardened limiter
image built from `/Users/futurepaul/dev/finite/finitecomputer`.

Finite Private rollout uses this published digest-pinned limiter image:

```text
ghcr.io/finitecomputer/finite-private-limiter:2026-06-18.deep-health.1@sha256:32d357c5d01bfa027381c07b357a4ed96602dabede5656b18907c53beaf16d18
```

Package visibility/access is confirmed: anonymous `docker buildx imagetools
inspect` succeeds against the pinned image.

The latest measured Kimi limiter release was
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

## GLM 5.2 Rollout

Use the Finite Private ops runbook in
`/Users/futurepaul/dev/finite/finitecomputer/docs/finite-private-ops.md`.

1. Confirm `tinfoil-config.yml` keeps the GLM/vLLM container on port `8001`,
   the limiter container on port `8002`, and `shim.upstream-port: 8002`.
2. Confirm the Tinfoil GLM image digest is pinned. Do not release a config that
   contains the all-zero placeholder digest from upstream source.
3. Trigger this repository's **Tinfoil Release** workflow with a new version tag
   such as `v2026-06-27-glm-5-2-limiter`.
4. Wait for the measured GitHub Release assets from
   `tinfoil-release-publish.yml`; do not relaunch from a plain tag without
   measured Tinfoil assets.
5. From the `finitecomputer` repo, relaunch the existing live Tinfoil
   deployment to the measured tag:
   `scripts/finite_private_ops.sh relaunch TAG`.
6. Run `scripts/finite_private_ops.sh wait-ready`, then
   `scripts/finite_private_ops.sh gate` from the `finitecomputer` repo.
7. Verify Core shows a settled canary reservation for `glm-5-2`, not a stuck
   reservation.

Stop the rollout if:

- the release lacks `tinfoil-deployment.json` or `tinfoil.hash`;
- Tinfoil cannot measure the GLM-plus-limiter config;
- `/health` does not report both upstream vLLM and Core usage API readiness;
- the authenticated canary does not settle in Core;
- relaunch would require changing the outer Tinfoil deployment name instead of
  relaunching the existing deployment.

Previous live rollout on 2026-06-18:

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

Keep a rollback commit that restores the last known-good Kimi config, or, in an
emergency only, restores `shim.upstream-port: 8001` for direct-vLLM recovery.
