# Grok through MixRoute: paired API audit

## Introduction

Compare MixRoute with direct xAI using identical requests. **Only Grok is tested: `grok-4.6` and `grok-4.7`. These results do not describe other MixRoute models.**

## Findings

MixRoute `grok-4.6` returned native calls despite `tool_choice: "none"` in **19/52** applicable responses; direct xAI: **0/52**. The focused test reproduced this in **11/20 pairs (55%)**. These frequencies describe selected tests, not all traffic.

Other failures: a JSON documentation example became an actual call; two calls appeared with parallel calls disabled; requested calls became text; `not_found` became `READY`. Contracts: [MixRoute](https://docs.mixroute.ai/en/api-reference/endpoint/chat-openai), [xAI](https://docs.x.ai/developers/tools/function-calling).

### Suspicious indicators

| Observation | Evidence and interpretation |
|---|---|
| Possible full-response caching | Six MixRoute 4.6 repetitions included **two identical complete-body replays** and another `chatcmpl-cache-*` response: 0.899–1.180 s versus 4.518 s initially. Direct xAI had neither marker nor body/ID replays. |
| `grok2api` fingerprints | **25/86** MixRoute 4.6 responses carried `x-grok2api-protocol`, with account, affinity, and prompt-stabilization headers. |
| Additional relay layer | `x-new-api-version`: **85/86** for MixRoute 4.6. `x-oneapi-request-id`: **85/86** for 4.6, **66/66** for 4.7. Neither appeared at xAI. Consistent with [New API or a compatible relay](https://github.com/QuantumNous/new-api/blob/789c970199ea527e6a26e071915f4a4cd2c64178/middleware/cors.go). |
| Possible tool emulation | Failed calls contained `endtool` markers or textual JSON. The [grok2api Web adapter](https://github.com/chenyme/grok2api/blob/5e5ad75556b61a2c4a8fcf344d83bfe7760f2b42/backend/internal/infra/provider/web/tools.go) emulates tools through text: a plausible mechanism, not proven attribution. |
| Self-identification | One 4.6 response said “Grok 4.5” while admitting its version was unverified. This is not attestation. |

Replayed bodies included identical IDs, timestamps, and usage despite `no-cache` headers. Equal text or `cached_tokens` alone is insufficient. Headers are server-controlled clues; proxying alone is not misconduct.

## Method and results

**28 September 2026: 152 local pairs / 304 requests.** Baseline: 60 pairs; expanded: 40; focused prohibition: 40; cache: 12. All outcomes retained.

| Endpoint / model | Requests | Pass | Fail | Inconclusive | Observation |
|---|---:|---:|---:|---:|---:|
| xAI / grok-4.6 | 86 | 82 | 0 | 0 | 4 |
| MixRoute / grok-4.6 | 86 | 49 | 34 | 1 | 2 |
| xAI / grok-4.7 | 66 | 64 | 0 | 0 | 2 |
| MixRoute / grok-4.7 | 66 | 63 | 1 | 0 | 2 |

The 4.7 failure was an exact-output instruction failure. Observations are unscored identity text; forbidden calls still fail.

Bodies are byte-identical per pair; xAI runs first. Synthetic tools never execute. UUIDs change except for cache tests. Node 26.8.2; no retries; 60-second timeout; 1,024 output tokens; low reasoning effort. Targeted cases, fixed order, and one account per provider limit generalization.

[Evidence](https://github.com/Heterotroph/FakeByMixRouter/releases/tag/paired-validation): raw transcripts, source snapshots, reports, checksums, and complete example pairs. Every verdict is reproducible. Secrets and account identifiers are redacted. These runs are local, not GitHub-hosted.

## Impact and possible conclusions

Missing calls leave work undone; unexpected calls can trigger unintended actions. Recovery adds latency and potentially costs. Replayed completions can defeat freshness requirements.

**Full-response caching suggests content retention and a potential conflict with MixRoute's zero-retention promise.** [Terms §7.2](https://docs.mixroute.ai/en/service-term) prohibit MixRoute's payload retention/caching; §7.3 permits upstream retention. Cache ownership, location, duration, storage medium, and retention of complete requests remain unknown. A breach by MixRoute is not yet established.

Fraud, substitution, subscription use, leakage, and incorrect billing are unproven. Questions: who retains content, for how long, and what is billed on replay?

## Run

Set GitHub secrets `XAI_API_KEY` and `MIXROUTE_API_KEY`; use [Actions → Run workflow](https://github.com/Heterotroph/FakeByMixRouter/actions/workflows/probe.yml). Runs are manual and publish evidence releases, including failed runs. Releases outlast log retention but remain owner-deletable.

With both keys in the local environment:

```sh
node probe.mjs --self-test
MODEL=grok-4.6 PAIRS=20 SCENARIOS=tool_choice_none,tool_choice_none_api_only node probe.mjs
MODEL=grok-4.6 PAIRS=6 SCENARIOS=cache_plain node probe.mjs
node probe.mjs --verify transcript.jsonl
```

Without `SCENARIOS`, all 22 cases run; `PAIRS=44` repeats them twice. Verification needs no keys or API calls. Exit codes: `0` no failures; `1` failures; `2` incomplete/inconclusive or setup error.
