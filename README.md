# bug-bounty-writeups

Public write-ups of my bug bounty findings, from recon to PoC.

Each folder is one finding: the full story of how it was found, the technique behind it, and the evidence. Targets are redacted where required; every claim is tied to a captured request, a screenshot, or a hash-verified artifact.

## What you'll find here

- **Write-ups** with the full journey: the first observation, how it turned into a finding, and a working, reproducible PoC
- **Real evidence** — screenshots, request captures, and artifacts, nothing staged
- **Honesty about outcomes** — accepted, duplicate, resolved, or rejected: it's all part of the story

## Write-ups

| # | Finding | Method | Outcome |
|---|---------|--------|---------|
| 1 | [The screenshot feature that called home: from a sign-up wizard to AWS metadata](ssrf-via-dns-rebind-on-scraper/writeup.md) | SSRF via DNS rebinding (TOCTOU) on a server-side screenshot scraper | Triaged P3 / Duplicate |
| 2 | [One link, two clicks, full account: open OAuth client registration on an MCP server](mcp-oauth-dcr-account-takeover/writeup.md) | Unauthenticated Dynamic Client Registration (RFC 7591) → spoofed-name consent → no-secret token redemption → 263 MCP tools | Triaged Duplicate |

*More coming as findings are cleared for disclosure.*

## How it's organized

One folder per finding, always the same shape:

```
bug-bounty-writeups/
└── <finding-name>/
    ├── writeup.md        ← the story + technical steps + PoC
    └── evidence/          ← screenshots and artifacts referenced in the write-up
```

---

*Findings are published only after the corresponding triage decision. No live secrets, credentials, or unredacted target names in these pages.*
