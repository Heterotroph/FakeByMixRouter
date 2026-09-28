# Grok through MixRoute: paired API audit

**Only Grok is tested: `grok-4.6` and `grok-4.7`. No conclusions about other MixRoute models.**

**Observed behavior:** I repeatedly observed Grok through MixRoute claiming to have completed actions that had not actually occurred. This observation is separate from the recorded paired tests below.

## Findings

1. **One wrong answer was replayed three times, byte for byte.** In [six GitHub requests](https://github.com/Heterotroph/FakeByMixRouter/actions/runs/36480113387), MixRoute `grok-4.6` returned an incorrect complete response four times with the same ID, timestamp, and usage, despite `no-cache` headers. Direct xAI had no body/ID replays. This suggests full-response retention somewhere on the route.
2. **Untrusted text became a real tool call, even with tools disabled.** In [GitHub pair 15](https://github.com/Heterotroph/FakeByMixRouter/actions/runs/36479076168), an instruction inside a failed tool result produced `create_checklist` despite `tool_choice: "none"`; xAI returned `NOT_FOUND`. A [separate tool-prohibition test](https://github.com/Heterotroph/FakeByMixRouter/actions/runs/36479787379), using different scenarios, produced forbidden calls in **12/20** MixRoute responses versus **0/20** xAI responses. No returned tool was executed.
3. **26 failures versus zero on identical Grok 4.6 requests.** Across 48 GitHub pairs, MixRoute failed 26 times and timed out once; direct xAI had neither. Failures included forbidden calls, textual imitations, and broken formats.

These frequencies describe selected tests, not all traffic. API contracts: [MixRoute](https://docs.mixroute.ai/en/api-reference/endpoint/chat-openai), [xAI](https://docs.x.ai/developers/tools/function-calling).

## Recorded GitHub runs

**Six completed runs, 96 pairs / 192 API requests.** Open **audit → Run paired requests and print raw evidence** in any linked run to inspect both providers' requests and responses. GitHub records the tested commit and execution times.

| Recorded run | Pairs | xAI failures | MixRoute failures |
|---|---:|---:|---:|
| [All cases · Grok 4.6](https://github.com/Heterotroph/FakeByMixRouter/actions/runs/36479076168) | 22 | 0 | 9 |
| [All cases · Grok 4.7](https://github.com/Heterotroph/FakeByMixRouter/actions/runs/36479081462) | 22 | 1 | 0 |
| [Tools disabled · Grok 4.6](https://github.com/Heterotroph/FakeByMixRouter/actions/runs/36479787379) | 20 | 0 | 12 |
| [Tools disabled · Grok 4.7](https://github.com/Heterotroph/FakeByMixRouter/actions/runs/36479555640) | 20 | 0 | 0 |
| [Cache · Grok 4.6](https://github.com/Heterotroph/FakeByMixRouter/actions/runs/36480113387) | 6 | 0 | 5 |
| [Cache · Grok 4.7](https://github.com/Heterotroph/FakeByMixRouter/actions/runs/36479878901) | 6 | 0 | 0 |

Additionally, the first run had one MixRoute timeout. The xAI 4.7 failure was an exact-output instruction failure. At most two workflows ran concurrently.

Each run published its transcript, report, and checksums in a [release](https://github.com/Heterotroph/FakeByMixRouter/releases). All six passed offline verification against the fixtures and checksums. Red runs indicate detected failures or incomplete responses; their evidence was preserved. These records establish API behavior, not upstream identity.

## Suspicious indicators (local baseline)

| Observation | Evidence and interpretation |
|---|---|
| `grok2api` fingerprints | **25/86** MixRoute 4.6 responses carried `x-grok2api-protocol`, with account, affinity, and prompt-stabilization headers. |
| Additional relay layer | `x-new-api-version`: **85/86** for MixRoute 4.6. `x-oneapi-request-id`: **85/86** for 4.6, **66/66** for 4.7. Neither appeared at xAI. Consistent with [New API or a compatible relay](https://github.com/QuantumNous/new-api/blob/789c970199ea527e6a26e071915f4a4cd2c64178/middleware/cors.go). |
| Possible tool emulation | Failed calls contained `endtool` markers or textual JSON. The [grok2api Web adapter](https://github.com/chenyme/grok2api/blob/5e5ad75556b61a2c4a8fcf344d83bfe7760f2b42/backend/internal/infra/provider/web/tools.go) emulates tools through text: a plausible mechanism, not proven attribution. |
| Self-identification | One 4.6 response said “Grok 4.5” while admitting its version was unverified. This is not attestation. |

The GitHub replays took 0.764–1.974 s versus 5.996 s initially. A separate local run also returned a `chatcmpl-cache-*` ID. Equal text or `cached_tokens` alone is insufficient. Headers are server-controlled clues; proxying alone is not misconduct.

## Local baseline and method

**28 September 2026: 152 local pairs / 304 requests.** Baseline: 60 pairs; expanded: 40; focused prohibition: 40; cache: 12. All outcomes retained.

| Endpoint / model | Requests | Pass | Fail | Inconclusive | Observation |
|---|---:|---:|---:|---:|---:|
| xAI / grok-4.6 | 86 | 82 | 0 | 0 | 4 |
| MixRoute / grok-4.6 | 86 | 49 | 34 | 1 | 2 |
| xAI / grok-4.7 | 66 | 64 | 0 | 0 | 2 |
| MixRoute / grok-4.7 | 66 | 63 | 1 | 0 | 2 |

The local 4.7 failure concerned exact output. Other local failures included `READY` after `not_found` and a JSON documentation example becoming a native call. Identity text is unscored; forbidden calls still fail.

Bodies are byte-identical per pair; xAI runs first. Synthetic tools never execute. UUIDs change except for cache tests. Node 26.8.2; no retries; 60-second timeout; 1,024 output tokens; low reasoning effort. Targeted cases, fixed order, and one account per provider limit generalization.

[Local evidence](https://github.com/Heterotroph/FakeByMixRouter/releases/tag/paired-validation): transcripts, source snapshots, reports, checksums, and example pairs. Secrets and account identifiers are redacted.

## Impact and possible conclusions

Missing calls leave work undone; unexpected calls can trigger unintended actions. Recovery adds latency and potentially costs. Replayed completions can defeat freshness requirements.

**Full-response caching suggests content retention and a potential conflict with MixRoute's zero-retention promise.** [Terms §7.2](https://docs.mixroute.ai/en/service-term) prohibit MixRoute's payload retention/caching; §7.3 permits upstream retention. Cache ownership, location, duration, storage medium, and retention of complete requests remain unknown. A breach by MixRoute is not yet established.

These anomalies raise suspicions of undisclosed routing, subscription-based access, model substitution, and data retention by MixRoute or its upstream providers. These explanations remain unproven, as do deliberate deception, data leakage, and incorrect billing. Establishing what happens internally is practically impossible from these API responses alone: it would require internal routing, account, retention, or billing records, or an independent audit with access to them. Questions: who retains content, for how long, and what is billed on replay?

## Run

Set GitHub secrets `XAI_API_KEY` and `MIXROUTE_API_KEY`; use [Actions → Run workflow](https://github.com/Heterotroph/FakeByMixRouter/actions/workflows/probe.yml). Runs publish evidence releases, which outlast log retention but remain owner-deletable.

With both keys in the local environment:

```sh
node probe.mjs --self-test
MODEL=grok-4.6 PAIRS=20 SCENARIOS=tool_choice_none,tool_choice_none_api_only node probe.mjs
MODEL=grok-4.6 PAIRS=6 SCENARIOS=cache_plain node probe.mjs
node probe.mjs --verify transcript.jsonl
```

Without `SCENARIOS`, all 22 cases run; `PAIRS=44` repeats them twice. Verification needs no keys or API calls. Exit codes: `0` no failures; `1` failures; `2` incomplete/inconclusive or setup error.
