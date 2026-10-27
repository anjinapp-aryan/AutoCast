# 04 — AI / LLM Dependency

Requirement: core publishing must work with **no LLM and no LLM API key**.

| Candidate | LLM needed for publishing? | Evidence | Notes |
|---|---|---|---|
| pendpost | **No** for publish path (not fully traced) | `package.json` has no runtime deps; platform logic in `scripts/*-social.mjs` calls platform APIs only | Designed for an external MCP agent. Contains optional `agent-runner.mjs`, `radar-*` modules (LLM judging; not traced). Approval is by a human. |
| SimplePost | **No** | `sdk/package.json` dependencies: googleapis, axios, sharp, @atproto/api, twitter-api-v2, AWS S3, zod… no LLM SDK | Agent skills/MCP are an interface layer. |
| Postiz | UNKNOWN (has AI features) | Not traced; AGPL so low priority | |
| Mixpost Lite | UNKNOWN | Not traced | |
| Open-Dispatch | No for dispatch; optional `ai/` module | `requirements.txt` has no LLM SDK | |
| RP-SMM | Only for comment replies | `ai_client.py` supports Anthropic/OpenAI/Gemini; `requirements.txt` has no LLM SDK (raw HTTP) | Not needed for publishing. REJECTed anyway (no license). |

## Conclusion

Two MIT candidates have an LLM-free publish path. The AutoCast design must keep AI behind an interface (`MetadataSuggester`) with a null implementation; AI output is only ever a **draft** that enters the same approval flow. No API key variables should be read by the core.
