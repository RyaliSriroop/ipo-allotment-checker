# IPO Allotment Group Checker — Project Specification

**Version:** 1.0  
**Status:** Implementation specification  
**Purpose:** Authoritative specification for development and review  
**V1 target:** A small private group, approximately 10–15 members per group, with a ₹0 infrastructure budget and no permanently running personal-laptop server.

---

## 1. Document Purpose

This is the authoritative engineering specification for the IPO Allotment Group Checker.

It must be understandable to:
- a new developer joining the project,
- a reviewer evaluating the architecture,
- and an AI coding agent such as Google Antigravity.

The application must be implemented as an independent project. The open-source project **IPO_Desk** is a technical reference for understanding registrar communication only. It is not the architectural, UI, database, or source-code template for this application.

Where this document marks an external integration as requiring verification, it must be verified before implementation. Do not invent endpoints, payloads, headers, response formats, or registrar behavior.

---

## 2. Product Goal

The problem is repetitive manual IPO allotment checking.

A group of approximately 10–15 people may apply for an IPO. Checking every person manually normally requires repeatedly:
1. opening an allotment-status website,
2. selecting the IPO,
3. entering a PAN,
4. completing any required verification,
5. checking the result,
6. repeating the process for every member.

The application reduces this to one group-level operation:

**Select IPO → Enter Group ID → Complete CAPTCHA → Check Status → Receive the entire group's results.**

The public user must see only:
- member name,
- normalized allotment status.

PAN numbers and other sensitive member information must never be exposed to group members.

---

## 3. Users and Access Model

### Manager

A manager owns exactly one group in V1.

The manager has an authenticated account and can:
- create/manage the group,
- add members,
- remove members,
- update member details,
- manage stored member PANs,
- access the manager dashboard,
- perform sensitive management operations only after PIN authorization.

The manager must not be able to access another manager's group.

### Group Member / Public Checker

A group member does not need an account.

They:
1. open the public checker,
2. select an IPO from the available IPO dropdown,
3. enter the Group ID,
4. complete CAPTCHA/human verification,
5. submit the check,
6. receive the group's results.

They must never receive:
- PAN numbers,
- manager credentials,
- manager PIN,
- encrypted PAN values,
- registrar request payloads,
- registrar raw responses,
- sensitive internal IDs.

---

## 4. End-to-End Workflows

### Manager workflow

```text
Manager
  ↓
Login
  ↓
Authenticated Manager Session
  ↓
Manager Dashboard
  ↓
Group Management
  ↓
Add / Update / Remove Members
  ↓
Enter Manager PIN for sensitive action
  ↓
Validate PIN
  ↓
Perform database operation
```

The manager should not have to enter the PIN for every single management action. A secure short-lived PIN authorization state may be used until logout/expiry.

### Public checker workflow

```text
Group Member
  ↓
Open public checker
  ↓
Select IPO
  ↓
Enter Group ID
  ↓
CAPTCHA
  ↓
POST /api/allotment/check
  ↓
Validate Group + IPO + CAPTCHA
  ↓
Load members and encrypted PANs server-side
  ↓
Identify official registrar
  ↓
Select registrar adapter
  ↓
Controlled async allotment engine
  ↓
Check applicable registrar source(s)
  ↓
Normalize results
  ↓
Store check result
  ↓
Return safe response
  ↓
Frontend displays Name + Status only
```

---

## 5. IPO Catalogue and Discovery

The backend maintains a central IPO catalogue containing at least:
- internal IPO ID,
- IPO name,
- registrar,
- registrar-specific IPO/client identifier,
- lifecycle/result status,
- allotment date where known,
- last synchronization timestamp,
- synchronization/source metadata.

Conceptually:

```text
IPO
 ├── id
 ├── name
 ├── registrar
 ├── registrar_ipo_id
 ├── status
 ├── allotment_date
 └── last_synced_at
```

The frontend displays human-readable IPO names. Registrar-specific identifiers stay backend-side.

IPO discovery should be independently refreshable per registrar:

```text
IPO Catalogue Sync
       │
 ┌─────┼──────────┐
 ↓     ↓          ↓
KFin   MUFG     Bigshare
 ↓     ↓          ↓
Normalize each source
       ↓
PostgreSQL catalogue
       ↓
Frontend IPO dropdown
```

Failure of one discovery source must not unnecessarily remove valid IPOs from other sources.

A short catalogue cache/TTL may be used, but PostgreSQL is the durable source. Do not depend on local disk/memory because V1 is intended for serverless deployment.

---

## 6. Registrar Integration Architecture

V1 supports:
1. **KFintech**
2. **MUFG Intime**
3. **Bigshare**

Use an adapter architecture:

```text
Allotment Engine
      ↓
Registrar Registry
      ↓
Correct Registrar Adapter
      ↓
Registrar HTTP interface
      ↓
Response
      ↓
Parser
      ↓
Normalized Result
```

The allotment engine must contain no registrar-specific HTTP logic.

Conceptual interface:

```typescript
interface RegistrarAdapter {
  getActiveIPOs(): Promise<RegistrarIPO[]>;
  checkAllotment(
    pan: string,
    registrarIpoId: string
  ): Promise<RegistrarResult>;
}
```

A bulk method may be added only if the registrar actually supports a legitimate bulk operation.

---

## 7. How Registrar Endpoints Work

This section is intentionally educational.

For every registrar, the adapter performs:

```text
Our backend receives:
PAN + IPO

        ↓

Determine IPO's official registrar

        ↓

Select registrar adapter

        ↓

Adapter builds registrar-specific HTTP request

        ↓

Our server sends request

        ↓

Registrar returns response
(JSON / HTML / other format)

        ↓

Adapter validates and parses response

        ↓

Adapter converts it to our standard result

        ↓

Allotment Engine receives normalized result
```

For each registrar, the implementation must document:
- verified endpoint,
- HTTP method,
- URL construction,
- query/body parameters,
- required headers,
- IPO/client identifier,
- response format,
- allotment field(s),
- status mapping,
- timeout behavior,
- retry behavior,
- known failures,
- whether CAPTCHA/user interaction is required,
- whether server-side access is legitimate.

**Do not guess any of these values.**

---

## 8. IPO_Desk Reference Policy

IPO_Desk is a reference implementation for registrar integration research only.

It may be inspected to understand:
- registrar integrations,
- endpoint patterns,
- request construction,
- response parsing,
- IPO discovery,
- caching/fault-isolation ideas.

The application must NOT:
- copy IPO_Desk's application architecture wholesale,
- copy its UI,
- copy its database design as-is,
- copy its business logic as-is,
- copy source code wholesale,
- assume README endpoint values are permanently valid.

Correct process:

```text
Study IPO_Desk
      ↓
Understand registrar communication
      ↓
Verify current/legitimate registrar behavior
      ↓
Document communication contract
      ↓
Implement our own adapter
      ↓
Normalize response
      ↓
Our application result
```

Do not bypass CAPTCHA, anti-bot systems, authentication, rate limits, or other access controls. If an integration cannot be legitimately automated, report it as unsupported instead of attempting a bypass.

---

## 9. KFintech Adapter

IPO_Desk currently documents a KFintech allotment endpoint pattern around:

```text
.../prod/api/query?type=pan
```

This is a reference point, not a guaranteed permanent contract.

Before implementation, inspect the relevant adapter and verify the current:
- endpoint,
- method,
- parameters,
- headers,
- payload,
- response,
- result mapping.

Conceptual flow:

```text
Allotment Engine
      ↓
KFintechAdapter
      ↓
Build verified KFintech request
      ↓
HTTP request
      ↓
KFintech response
      ↓
Validate + parse
      ↓
RegistrarResult
```

---

## 10. MUFG Intime Adapter

MUFG Intime is the V1 provider for the MUFG/Link Intime family.

IPO_Desk documents an allotment operation around:

```text
/Initial_Offer/IPO.aspx/SearchOnPan
```

The exact request must be obtained from the verified implementation/current behavior rather than guessed.

Conceptual flow:

```text
Allotment Engine
      ↓
MUFGAdapter
      ↓
Build verified MUFG request
      ↓
HTTP request
      ↓
MUFG response
      ↓
Validate + parse
      ↓
RegistrarResult
```

If no valid application record exists, the adapter may return `NOT_FOUND`.

If the response schema is malformed or the expected allotment field cannot be identified, return an error/unknown state rather than guessing.

---

## 11. Bigshare Adapter

IPO_Desk documents Bigshare discovery around:

```text
https://ipo.bigshareonline.com/IPO_Status.html
```

and an allotment operation around:

```text
/Data.aspx/FetchIpodetails
```

The exact request payload and response mapping must be verified from the actual implementation/current behavior before use.

The documented interpretation includes:
- positive allotted quantity → allotted,
- zero allotted quantity → not allotted,
- no valid application record → not found,
- malformed response → error.

If multiple legitimate Bigshare mirrors/endpoints are used, they must be independently verified and must not be queried blindly.

---

## 12. Registrar Selection and Fallback

The application must NOT automatically check every IPO against every registrar.

First:

```text
IPO
 ↓
Official registrar
 ↓
Primary adapter
```

Do not blindly do:

```text
KFintech
 ↓
MUFG
 ↓
Bigshare
```

because the other providers may not contain the application's record for that IPO.

### Validated fallback

If the primary source returns `NOT_FOUND`:

```text
NOT_FOUND
   ↓
Is there a verified applicable fallback?
   ├── No → APPLICATION_NOT_FOUND
   └── Yes
          ↓
      Check fallback
```

If fallback finds:
- `ALLOTTED` → final `ALLOTTED`
- `NOT_ALLOTTED` → final `NOT_ALLOTTED`
- `NOT_FOUND` → continue only through other verified applicable sources.

If all legitimate applicable sources return `NOT_FOUND`:

```text
APPLICATION_NOT_FOUND
```

A registrar/network failure must NEVER be interpreted as `NOT_FOUND`.

---

## 13. Result States

Internal states:

```text
FOUND_ALLOTTED
FOUND_NOT_ALLOTTED
NOT_FOUND
RESULT_NOT_AVAILABLE
TEMPORARY_ERROR
INVALID_RESPONSE
```

User-facing states:

```text
ALLOTTED
NOT_ALLOTTED
APPLICATION_NOT_FOUND
RESULT_NOT_AVAILABLE
CHECK_TEMPORARILY_UNAVAILABLE
```

Rules:
- `RESULT_NOT_AVAILABLE` is never `APPLICATION_NOT_FOUND`.
- `TEMPORARY_ERROR` is never `APPLICATION_NOT_FOUND`.
- `INVALID_RESPONSE` is never `APPLICATION_NOT_FOUND`.
- `NOT_FOUND` is never the same as `NOT_ALLOTTED`.

---

## 14. Allotment Engine

The allotment engine receives a group and IPO, loads member PANs server-side, and checks them using controlled asynchronous concurrency.

For approximately 10–15 members:

```text
12 members
    ↓
Controlled Worker Pool
    ↓
┌────┬────┬────┬────┐
│ W1 │ W2 │ W3 │ W4 │
└────┴────┴────┴────┘
    ↓
Registrar requests
    ↓
Individual normalized results
```

Start with approximately 3–5 concurrent requests as a configurable value. Do not create unnecessary OS-level threads.

The goal is:

**reasonable parallelism + registrar-friendly request rate + reliable results.**

Each member's flow:

```text
Member
 ↓
Decrypt PAN only server-side when needed
 ↓
Primary registrar
 ↓
Normalize
 ↓
Validated fallback if applicable
 ↓
Final status
```

---

## 15. Result Consistency

Multiple sources must never be used to arbitrarily overwrite one another.

Rules:
1. Confirmed `ALLOTTED` is authoritative for that source.
2. Confirmed `NOT_ALLOTTED` is authoritative for that source.
3. `NOT_FOUND` is not `NOT_ALLOTTED`.
4. Errors are not `NOT_FOUND`.
5. Fallback results record their source.
6. Conflicting legitimate results must be flagged, not silently resolved.
7. The application must never fabricate a definitive result.

Store safe metadata such as:
- source registrar,
- check time,
- internal result state,
- final normalized state.

Do not store raw registrar responses unless specifically justified by a secure retention/debugging requirement.

---

## 16. Public API

Conceptual endpoint:

```http
POST /api/allotment/check
```

Request:

```json
{
  "groupId": "GRP-A7X92",
  "ipoId": "IPO123",
  "captchaToken": "..."
}
```

Backend:
1. validates request,
2. verifies CAPTCHA,
3. validates group,
4. validates IPO,
5. loads members,
6. determines registrar,
7. executes allotment engine,
8. stores result,
9. returns safe data.

Example response:

```json
{
  "ipo": "ABC Technologies",
  "results": [
    {
      "name": "Rahul",
      "status": "ALLOTTED"
    },
    {
      "name": "Arun",
      "status": "NOT_ALLOTTED"
    },
    {
      "name": "Karthik",
      "status": "APPLICATION_NOT_FOUND"
    }
  ]
}
```

PAN must never appear in this response.

If a member's application is not found after all applicable verified checks, display `APPLICATION_NOT_FOUND`.

---

## 17. Manager APIs

Conceptual endpoints:

```http
POST   /api/manager/members
GET    /api/manager/members
PUT    /api/manager/members/:id
DELETE /api/manager/members/:id
```

Every sensitive endpoint requires:
- authenticated manager session,
- authorization to that manager's group,
- valid PIN authorization.

Prevent IDOR-style access between groups.

---

## 18. Database

Use **Supabase PostgreSQL** as the hosted database and **Prisma** as the ORM.

Conceptual relationship:

```text
MANAGER
   │
   │ 1:1
   ↓
GROUP
   │
   │ 1:N
   ↓
MEMBER

IPO
 │
 └── REGISTRAR

MEMBER ─── ALLOTMENT_CHECK ─── IPO
```

Suggested core tables:

### managers
- id
- auth_user_id
- email
- created_at
- updated_at

### groups
- id
- manager_id
- group_code
- name
- created_at
- updated_at

### members
- id
- group_id
- name
- encrypted_pan
- active
- created_at
- updated_at

### ipos
- id
- name
- registrar
- registrar_ipo_id
- status
- allotment_date
- last_synced_at
- created_at
- updated_at

### allotment_checks
- id
- member_id
- ipo_id
- final_status
- source_registrar
- checked_at
- safe diagnostic metadata where needed

The final schema must add appropriate primary keys, foreign keys, unique constraints and indexes.

---

## 19. PAN Security

PAN is sensitive personal data in this application.

Required flow:

```text
PAN entered by manager
        ↓
Validate
        ↓
Encrypt
        ↓
Store encrypted value
```

During a check:

```text
Encrypted PAN
      ↓
Decrypt only server-side
      ↓
Registrar adapter
      ↓
Do not expose/log plaintext
```

Never:
- log PANs,
- return PANs publicly,
- put PANs in URLs,
- put PANs in browser local storage,
- include PANs in analytics,
- commit PANs to Git.

Encryption keys must be deployment secrets/environment variables.

---

## 20. Authentication and Manager PIN

Use **Supabase Auth** for manager authentication.

V1 mapping:

```text
One Manager Account
        ↓
One Group
```

The manager PIN is separate from login credentials and protects sensitive management actions.

The PIN must:
- never be stored plaintext,
- be hashed with a modern password/PIN hashing algorithm,
- be verified server-side,
- never be sent to the client for comparison.

Use a secure short-lived PIN authorization state so the manager does not have to enter the PIN for every individual add/update/remove action.

---

## 21. CAPTCHA and Public Abuse Protection

Use a free CAPTCHA/human-verification service only if its current free tier and terms are suitable.

Preferred candidate:

**Cloudflare Turnstile.**

Flow:

```text
User
 ↓
CAPTCHA
 ↓
Token
 ↓
Backend
 ↓
Server-side verification
 ↓
Allotment check
```

The application's CAPTCHA is for protecting our public endpoint. It does not authorize bypassing a registrar's own CAPTCHA or anti-bot controls.

If Turnstile is not suitable, use strong server-side rate limiting and document the limitation.

---

## 22. Rate Limiting and Request Control

The public check endpoint must be rate limited.

Manager endpoints must also have suitable limits.

A malicious user must not be able to repeatedly trigger 10–15 registrar requests without control.

Consider caching recent completed group/IPO checks to reduce unnecessary registrar traffic, provided results are timestamped and the UI communicates freshness appropriately.

---

## 23. Notifications

V1 must NOT depend on a permanently running notification agent.

The personal laptop must not remain online.

When a user checks:

```text
User checks IPO
 ↓
Allotment Engine
 ↓
Results
 ↓
Store result
 ↓
Optional email notification
```

If email is implemented:
- use a free-tier provider suitable for actual volume,
- limit notifications to avoid spam,
- consider only IPO/member relationships relevant to stored application data,
- do not ask public users to manually enter each member's IPO applications if the system can determine the relationship from its data,
- never include PANs.

A reasonable V1 limit is 2–3 notifications per member/IPO event unless the chosen provider and requirements justify otherwise.

Continuous automated result monitoring is out of scope for V1.

---

## 24. Frontend

Use **React + Vite + Tailwind CSS**.

Public UI:
- home/check page,
- IPO selector,
- Group ID input,
- CAPTCHA,
- check button,
- loading/progress state,
- result table,
- clear error/result-not-available messages.

Manager UI:
- login,
- dashboard,
- group information,
- member list,
- add member,
- update member,
- remove member,
- PIN verification,
- logout.

Public results must contain only:

```text
Name | Status
```

---

## 25. Backend

Use **Node.js + Express + TypeScript**.

Recommended conceptual structure:

```text
backend/
└── src/
    ├── controllers/
    ├── routes/
    ├── services/
    ├── middleware/
    ├── validation/
    ├── security/
    ├── registrars/
    │   ├── RegistrarAdapter.ts
    │   ├── KFintechAdapter.ts
    │   ├── MUFGAdapter.ts
    │   └── BigshareAdapter.ts
    ├── allotment/
    │   ├── allotmentEngine.ts
    │   ├── workerPool.ts
    │   └── resultResolver.ts
    ├── ipo/
    │   ├── ipoCatalogue.ts
    │   └── ipoSync.ts
    └── database/
```

The exact structure may be refined, but registrar-specific HTTP logic must remain isolated.

---

## 26. Validation and Error Handling

Use **Zod** for request validation.

Use consistent safe error responses, for example:

```text
INVALID_GROUP
INVALID_IPO
CAPTCHA_FAILED
UNAUTHORIZED
INVALID_PIN
RATE_LIMITED
RESULT_NOT_AVAILABLE
CHECK_TEMPORARILY_UNAVAILABLE
```

Never leak:
- PAN,
- stack traces,
- database internals,
- secrets,
- raw upstream responses.

---

## 27. Technology Stack

| Layer | Technology |
|---|---|
| Frontend | React + Vite |
| Styling | Tailwind CSS |
| Backend | Node.js + Express |
| Language | TypeScript |
| Database | Supabase PostgreSQL |
| ORM | Prisma |
| Authentication | Supabase Auth |
| HTTP | Native `fetch` or Axios |
| Validation | Zod |
| Concurrency | Node.js async/Promise controlled worker pool |
| CAPTCHA | Cloudflare Turnstile if suitable |
| Rate limiting | express-rate-limit or equivalent |
| Testing | Vitest + Supertest |
| Source control | Git + GitHub |
| Development | VS Code + Google Antigravity |
| Deployment | Vercel + Supabase, subject to current free-tier suitability |

Do not add technologies merely for complexity or because they are popular.

---

## 28. ₹0 Infrastructure Requirement

V1 must operate without paid infrastructure.

Constraints:
- no paid server,
- no permanently running personal-laptop server,
- laptop is for development/testing only,
- no permanent background worker,
- free tiers must be sufficient for expected usage,
- no paid API dependency,
- free-tier limits must not be silently assumed to be unlimited.

Potential architecture:

```text
Internet
   ↓
Vercel
 ┌─┴───────────────┐
 ↓                 ↓
React UI       API/serverless functions
                    ↓
              Supabase PostgreSQL
                    ↓
              Registrar interfaces
```

Before deployment, verify current free-tier limits and terms of each selected service.

---

## 29. Environment Variables and Secrets

Use local:

```text
.env
```

and commit:

```text
.env.example
```

only with placeholders.

Possible variables:

```env
DATABASE_URL=
SUPABASE_URL=
SUPABASE_ANON_KEY=
SUPABASE_SERVICE_ROLE_KEY=
PAN_ENCRYPTION_KEY=
TURNSTILE_SITE_KEY=
TURNSTILE_SECRET_KEY=
EMAIL_PROVIDER_KEY=
```

Only retain variables actually needed by the final implementation.

Never commit real secrets.

Server-only keys must never be exposed to the frontend.

---

## 30. Testing Strategy

### Unit tests
Cover:
- PAN validation,
- encryption/decryption,
- PIN hashing/verification,
- result normalization,
- fallback rules,
- result precedence,
- worker-pool behavior,
- timeout handling,
- registrar response parsing.

### Integration tests
Cover:
- manager authentication,
- authorization,
- CRUD,
- public checker,
- database operations,
- CAPTCHA verification abstraction.

### Registrar adapter tests
Use mocked/recorded responses where appropriate.

Cover:
- allotted,
- not allotted,
- not found,
- result not available,
- malformed response,
- timeout,
- HTTP failure.

Do not hammer live registrar services during automated tests.

### End-to-end
Test:

```text
Manager creates group
 ↓
Manager adds members
 ↓
Public user selects IPO
 ↓
Public user enters Group ID
 ↓
CAPTCHA succeeds
 ↓
Engine checks members
 ↓
Results returned
 ↓
Only names + statuses displayed
```

---

## 31. Reliability

The system must favor correctness over speed.

A slower verified result is better than a fast incorrect result.

Never convert:
- timeout → not found,
- server error → not found,
- malformed response → not found,
- result not available → not found.

Every upstream call needs:
- timeout,
- controlled retry where safe,
- error classification,
- sensitive-data-safe logging.

---

## 32. Performance

For 10–15 members, the application should substantially reduce the time required compared with sequential manual checking.

The goal is not maximum concurrency.

The goal is:

**reasonable parallelism + registrar-friendly traffic + reliable results.**

Start with configurable concurrency around 3–5 and tune downward if legitimate registrar behavior requires it.

---

## 33. Project Structure

Antigravity should create the actual structure after reviewing this specification.

Conceptually:

```text
ipo-allotment-checker/
│
├── apps/
│   ├── web/
│   └── api/
│
├── prisma/
│   ├── schema.prisma
│   └── migrations/
│
├── docs/
│   └── PROJECT_SPECIFICATION.md
│
├── tests/
│
├── .env.example
├── .gitignore
├── package.json
└── README.md
```

The final structure may be simplified if that improves maintainability.

---

## 34. Development Workflow

```text
Specification
      ↓
Investigation
      ↓
Architecture confirmation
      ↓
Database design
      ↓
Implementation
      ↓
Tests
      ↓
Local run
      ↓
Manual verification
      ↓
Security review
      ↓
Git commit
      ↓
GitHub
```

Use incremental Git commits. Do not create one enormous unreviewed commit for the whole project.

---

## 35. Antigravity Implementation Instructions

Antigravity is the development agent, not a production application component.

### Phase 1 — Understand

Read this entire document before coding.

### Phase 2 — Investigate

Inspect IPO_Desk only for registrar integration knowledge.

Determine:
- actual adapter files,
- current request construction,
- response parsing,
- discovery mechanisms,
- endpoint patterns,
- assumptions.

Do not copy the project wholesale.

### Phase 3 — Verify

Verify registrar assumptions against the actual implementation/current legitimate behavior.

If an interface is unavailable, changed, protected, or cannot legitimately be automated, report it.

### Phase 4 — Design

Produce:
- architecture,
- database schema,
- API design,
- directory structure,
- registrar adapter design,
- allotment engine design,
- security design,
- testing plan,
- deployment plan.

### Phase 5 — Approval checkpoint

Before major implementation, summarize:
- confirmed facts,
- assumptions,
- unresolved issues,
- proposed decisions.

Do not silently make critical architectural decisions that contradict this specification.

### Phase 6 — Implement

Implement a new codebase using our own architecture and naming.

### Phase 7 — Test

Run:
- lint,
- unit tests,
- integration tests,
- build,
- local smoke tests.

### Phase 8 — Audit

Compare implementation against every requirement in this document.

### Phase 9 — Report

Provide:
- what was implemented,
- what was tested,
- what remains unresolved,
- registrar limitations,
- commands to run,
- environment variables required.

---

## 36. Explicit Anti-Assumption Rules

Antigravity must NOT:
- invent registrar APIs,
- invent request payloads,
- invent response fields,
- assume endpoints are permanent,
- treat HTTP errors as application-not-found,
- treat upstream failures as not-allotted,
- query every registrar for every IPO without evidence,
- bypass CAPTCHA,
- bypass anti-bot mechanisms,
- bypass authentication,
- bypass rate limits,
- expose PANs,
- log PANs,
- commit secrets,
- store plaintext PINs,
- create a permanent server requirement,
- add an LLM merely to label the application "agentic AI."

If something cannot be implemented reliably and legitimately, stop at that boundary and report the limitation.

---

## 37. What This Application Is

This application is:

**An automated multi-registrar IPO allotment checking system for a small private group.**

Its automation consists primarily of:

```text
Database
+
HTTP communication
+
Registrar adapters
+
Controlled async concurrency
+
Result normalization
+
Authentication/security
```

It does not require an LLM.

Antigravity being an agentic development tool does not make the resulting application an AI application.

---

## 38. Acceptance Criteria

V1 is complete only when:

### Manager
- manager can authenticate,
- one manager maps to one group,
- manager can add/update/remove members,
- sensitive actions require PIN authorization,
- manager cannot access another group.

### Public checker
- no account required,
- IPO can be selected,
- Group ID can be entered,
- CAPTCHA is enforced if enabled,
- one group-level check returns all active members,
- only name + status are displayed.

### Allotment engine
- correct registrar is selected,
- registrar logic is isolated in adapters,
- requests use controlled async concurrency,
- timeouts exist,
- errors are classified correctly,
- fallback occurs only when validated,
- no false application-not-found result is created from technical failure.

### Security
- PAN encrypted at rest,
- PAN never returned publicly,
- PAN never logged,
- PIN/password never stored plaintext,
- secrets excluded from Git,
- public endpoint rate limited,
- manager authorization enforced server-side.

### Infrastructure
- works locally,
- uses Supabase PostgreSQL,
- production does not require the developer's laptop online,
- free-tier suitability is documented.

### Quality
- critical business logic is tested,
- application builds,
- README has setup instructions,
- `.env.example` exists,
- no unresolved critical registrar assumption is hidden.

---

## 39. Known Limitations and Future Extensions

V1 intentionally excludes:
- permanent 24/7 IPO-result monitoring,
- guaranteed instant notifications,
- LLM/AI functionality,
- unlimited public usage,
- unlimited registrar requests,
- guaranteed registrar availability,
- bypass-based integrations.

Future possibilities:
- scheduled result monitoring,
- reliable notification service,
- email history,
- additional registrars,
- manager audit history,
- multiple groups,
- analytics,
- optional AI explanations.

---

## 40. Final Engineering Principle

**Optimize for correct, explainable, privacy-preserving results—not merely the fastest-looking answer.**

The system must distinguish between:

```text
ALLOTTED
NOT_ALLOTTED
APPLICATION_NOT_FOUND
RESULT_NOT_AVAILABLE
CHECK_TEMPORARILY_UNAVAILABLE
```

The registrar layer must be replaceable, PAN data must be protected, public responses must contain only safe information, and the application must remain simple enough to operate within the V1 ₹0 infrastructure constraint.
