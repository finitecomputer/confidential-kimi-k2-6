# confidential-kimi-k2-6

Current `tinfoil-config.yml` routes the public shim directly to vLLM on port
8001.

Finite Private rollout should use this published digest-pinned limiter image:

```text
ghcr.io/finitecomputer/finite-private-limiter:2026-05-26.stream-audit.1@sha256:f113a1b31dd3b25c7ac3978376ba3a263eb0c4f56072bf268aba3f1df6027a15
```

Package visibility/access is now confirmed: anonymous `docker buildx
imagetools inspect` succeeds against the pinned image. Then:

1. Add a `finite-private-limiter` container listening on `8002`.
2. Set limiter env:
   - `FINITE_USAGE_API_URL=https://finite.computer`
   - `UPSTREAM_BASE_URL=http://127.0.0.1:8001`
   - `DASHBOARD_URL=https://finite.computer/dashboard`
   - `LISTEN_ADDR=0.0.0.0:8002`
3. Add Tinfoil secrets:
   - `FINITE_USAGE_API_SERVICE_KEY`
   - `VLLM_INTERNAL_API_KEY`
   - `VLLM_API_KEY`
4. Set `VLLM_API_KEY` and `VLLM_INTERNAL_API_KEY` to the same value for the
   first limiter rollout.
5. Move `shim.upstream-port` from `8001` to `8002`.
6. Set `shim.authenticated: false` or remove the authenticated inference
   endpoint list so caller auth is the Finite Private `fpk_live_...` key
   validated by the limiter.

Keep a rollback commit that restores `shim.upstream-port: 8001` until the
limiter path has completed non-streaming and streaming canaries.
