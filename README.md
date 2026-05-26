# confidential-kimi-k2-6

Current `tinfoil-config.yml` routes the public shim to the Finite Private
limiter on port `8002`; the limiter forwards admitted requests to vLLM on
`8001` using the internal vLLM API key.

Finite Private rollout uses this published digest-pinned limiter image:

```text
ghcr.io/finitecomputer/finite-private-limiter:2026-05-26.stream-audit.1@sha256:f113a1b31dd3b25c7ac3978376ba3a263eb0c4f56072bf268aba3f1df6027a15
```

Package visibility/access is confirmed: anonymous `docker buildx imagetools
inspect` succeeds against the pinned image.

The measured release is
`https://github.com/finitecomputer/confidential-kimi-k2-6/releases/tag/v2026-05-26-private-limiter`
with deployment digest:

```text
2326cc66d33b7dac16cd1b424a8fed221411e28924ff1036ede9feb0fa1b09f6
```

Required Tinfoil secrets:

- `FINITE_USAGE_API_SERVICE_KEY`
- `VLLM_INTERNAL_API_KEY`
- `VLLM_API_KEY`

Set `VLLM_API_KEY` and `VLLM_INTERNAL_API_KEY` to the same value for the first
limiter rollout.

Keep a rollback commit that restores `shim.upstream-port: 8001` until the
limiter path has completed non-streaming and streaming canaries.
