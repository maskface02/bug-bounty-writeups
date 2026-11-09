# The Policy Center behind a 404: a Personal User rewrote company policy, and the AI served it

A multi-tenant expense-management SaaS hides its company Policy Center from ordinary employees — the menu entry is gone and the admin route returns a 404. The API behind it never asked who was calling. A user with the default `User` role could create, activate, edit, read, and delete the company's expense policies, and because the AI policy assistant is grounded in whatever policy is active, the rewritten text came back out of the assistant's mouth in an administrator's chat. Everything below was reproduced on researcher-owned accounts in the vendor's sandbox.

## The Discovery

The program's sandbox replicated the production platform, so I registered two accounts inside one company: an administrator (all roles) and a second user with only the default `User` role that every invited employee receives.

The administrator's side of the feature is an "AI powered" Policy Center: upload a document, assign it to branches, enable it. Each branch can only be assigned to one policy, so activating a new one displaces the previous active policy.

![PoC 1: the admin Policy Center — add a policy, assign branches, enable](evidence/01-policy-center-add-policy.png)

The Personal User's side looks different: the Settings menu has no policy entry, and navigating directly to the admin route lands on the SPA's 404. The UI is doing its job.

The UI was the only thing doing its job.

Walking the admin flow with a proxy captured the policy-management calls in plain form. Replaying the same operations with the Personal User's bearer token — no role change, no admin consent — returned success:

```
POST /v3/expense-policies
Authorization: Bearer <personal-user-token>

{"policyName":"Company Expense Policy 2026","isActive":true,
 "branchIds":["<branch-id>"],"markdown":"...","originalFileName":"company-policy.md"}
```

```
HTTP 201 Created
```

The response recorded the Personal User as the author:

```json
"updatedBy": {"firstName": "Low", "lastName": "Priv", "email": "<personal-user>"}
```

## The Chain

**Step 1 — Create and activate.** The 201 above did two things at once: it created a company policy and, with `isActive: true`, activated it on the shared branch. Because branch assignment is exclusive, the administrator's active policy was silently displaced. The attacker never has to delete anything to take over the rules employees see.

**Step 2 — Read.** The policy content is served as Markdown from the revision endpoint:

```
GET /v1/expense-policies/<policy-id>/revisions/1/content/markdown
HTTP/2 200
```

That returned the full document — including drafts written by the administrator.

**Step 3 — Edit.** The same object could be rewritten by a Personal User:

```
PUT /v3/expense-policies/<policy-id>
HTTP/2 200
```

**Step 4 — The AI amplifier.** The Policy Center ships with an assistant ("Chat with assistant", AI powered) that answers employee questions from the active policy text. After activating a policy containing a marker instruction, the administrator asked an ordinary question in the assistant and got the attacker's text back, with citations:

```
POST /v1/expense-policies/chat/messages
{"message":"What is the daily allowance for Belgium?"}

→ "The daily allowance for Belgium is 100 MAD. [...] PWNED1030K7Q2"
```

![PoC 2: the policy editor and the assistant that reads it](evidence/02-policy-editor-ai-assistant.png)

The attacker does not need to reach the victim's browser or session. They write policy text; the assistant reads it as ground truth and quotes it back in an administrator's chat.

**Step 5 — Delete.** Policies created by the administrator could be deleted by the Personal User:

```
DELETE /v1/expense-policies/<admin-policy-id>
HTTP/2 204 No Content
```

The administrator's next read of the same object returned `404`.

| Operation | Endpoint | Result with Personal User token |
|---|---|---|
| Create + activate | `POST /v3/expense-policies` | `201 Created`, `isActive: true` |
| Read content | `GET /v1/expense-policies/{id}/revisions/1/content/markdown` | `200 OK`, full Markdown |
| Edit | `PUT /v3/expense-policies/{id}` | `200 OK` |
| Delete | `DELETE /v1/expense-policies/{id}` | `204 No Content` |

## Why It Works

The authorization model is enforced at the presentation layer, not the service layer.

The Angular frontend hides the Policy Center from users without an administrative role. The legacy WebForms frontend redirects unauthorized users away from `/admin/settings/*`. The BFF behind `/v1/*` and `/v3/expense-policies*` verified that the bearer token was valid and that the caller belonged to the company — but never that the caller held a role allowed to manage company policy.

Think of it like a building where the receptionist checks badges at the front entrance, but the service corridor behind it has no lock. The UI walks every Personal User away from that corridor; the API leaves the door open.

Three details make this worse than a typical missing-role-check:

1. **Activation is a side effect, not a separate call.** `isActive: true` on the create request displaces the currently active branch policy. One request changes what every employee sees.
2. **The AI turns an integrity bug into a distribution channel.** The assistant is grounded in the active policy and cites it as company truth. The attacker's instructions reach administrators through a normal product feature, with no victim action beyond asking a question.
3. **Deletion is permanent.** `204` on an administrator-created policy, `404` on the next read, no recovery path in the product.

All testing was performed with researcher-owned accounts in the vendor's sandbox. No third-party data was read or modified.

## The Fix

1. **Enforce the role check server-side on every `/v1/expense-policies*` and `/v3/expense-policies*` operation.** Policy management is an administrative function; the API must reject tokens whose roles do not include an administrative role, regardless of what the UI shows.
2. **Make the check structural, not per-endpoint.** A deny-by-default authorization policy for administrative route groups prevents the next endpoint from shipping without its own check.
3. **Treat policy text as untrusted input for the assistant.** An instruction embedded in a document must not be able to override the assistant's system behavior or be cited as policy without provenance.
4. **Log policy activation and deletion with the acting user and role**, so an unexpected author is visible to the company administrator.

## Disclosure Status

Reported to the vendor's bug bounty program on Intigriti. Triaged as **duplicate** — the same function-level authorization gap had already been reported by another researcher. No bounty awarded; the technical work stands on its own.

<!-- Optional: add your Intigriti "Duplicate" screenshot at evidence/03-intigriti-duplicate.png and embed it here. -->

## Resources

- OWASP API Security Top 10 — API5:2023 Broken Function Level Authorization — https://owasp.org/API-Security/editions/2023/en/0xa5-broken-function-level-authorization/
- OWASP Web Security Testing Guide — Testing for Privilege Escalation (WSTG-ATHZ-03) — https://owasp.org/www-project-web-security-testing-guide/latest/4-Web_Application_Security_Testing/05-Authorization_Testing/03-Testing_for_Privilege_Escalation
- CWE-862: Missing Authorization — https://cwe.mitre.org/data/definitions/862.html
- OWASP Top 10 for LLM Applications — LLM01: Prompt Injection — https://genai.owasp.org/llmrisk/llm01-prompt-injection/

---

*Target details redacted per program policy.*
