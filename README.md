# KrosAI Documentation Audit

KrosAI’s public OpenAPI spec doesn’t match the human docs in key areas like auth, endpoint structure, request bodies, and webhooks. On top of that, five default GitBook tutorial pages are still sitting in the navigation, and the Help Center is just empty categories with no real articles.

The result is a set of small but compounding inconsistencies, enough that a developer (or an agent following the docs) is likely to run into basic failures without realizing why.


---

### 1. Three incompatible webhook event lists across three pages (critical)

**Location:** `/webhooks/overview`, `/webhooks/events`, `/api-reference/reference/webhooks`

**Problem:** Three "authoritative" surfaces describe three different event vocabularies:
- Overview lists: `call.started`, `call.ended`, `call.failed`, `call.recording.completed`, `transcription.completed`.
- Events page lists: `call.initiated`, `call.ringing`, `call.answered`, `call.completed`, `call.failed`, plus `recording.ready`, `transcript.ready`, `phone_number.*`, `endpoint.*`, `port_request.*`, `balance.low`, `payment.received` — none of `call.started`, `call.ended`, or `call.recording.completed` appear.
- OpenAPI `WebhookEvent` enum: `["call.started","call.completed","call.failed","call.recording.ready","transcription.completed","number.provisioned","number.released"]` — introduces `number.provisioned`/`number.released` which appear nowhere else, and renames recording to `.ready` (conflicting with both `.completed` and the Events page's bare `recording.ready`).

**Consequence:** A developer who subscribes to `call.ended` (per Overview) gets a `400` from the API because the enum requires `call.completed`. An agent indexing the Events page will register handlers (`phone_number.*`, `balance.low`) that the backend will never emit. Either way, webhooks silently never fire and debugging is hours of guesswork.

**The fix:** Designate the OpenAPI enum as source of truth, regenerate the Overview and Events tables from it, deprecate the names not present in the enum, and document `number.provisioned`/`number.released` explicitly (or remove them from the schema).

---

### 2. Auth scheme contradicts itself between docs and API reference (critical)

**Location:** `/getting-started/authentication`, `/api-reference/reference/phone-numbers`, `/getting-started-old/quickstart-1`

**Problem:** The Authentication page says: "Include your API key in the `x-api-key` header of every request." Every rendered endpoint in the API Reference inverts this: "Authorization string Required," with the note "Also accepts JWT tokens from Supabase Auth for dashboard sessions," and labels x-api-key as the "Alternative." The legacy `getting-started-old/quickstart-1` page (still live) uses Bearer-style auth in its prerequisites/samples. The current `getting-started/quickstart` uses `x-api-key`.

**Consequence:** A developer following the Authentication page sends only `x-api-key`, then opens the reference, sees `Authorization` marked "Required," and assumes their config is wrong. An agent extracting "required headers" from the API reference will hardcode `Authorization: Bearer …` and treat x-api-key as optional — the opposite of the documented primary path.

**The fix:** Pick one canonical header, mark the other "alternative (deprecated)" consistently in every page, and regenerate the API reference's per-endpoint auth block from a single shared component.

---

### 3. Endpoint type enum is incomplete in user docs (critical)

**Location:** `/voice/endpoints` vs `/api-reference/reference/endpoints`

**Problem:** The `/voice/endpoints` page says endpoint types are exactly two values: `agent` or `webhook`. The OpenAPI schema actually supports nine types: `webhook`, `agent`, `vapi`, `retell`, `livekit`, `elevenlabs`, `vogent`, `custom_sip`, and `direct_sip`. The user docs present `agent` as the only way to connect an AI voice provider, which means developers never learn about the provider-specific types (`vapi`, `retell`, `elevenlabs`, etc.) that most of them will actually need.

The per-provider config objects also require specific field names the user docs don't mention. ElevenLabs requires `elevenlabs_agent_id`, Vapi requires `assistant_id`, Retell requires `agent_id`, and LiveKit requires both `livekit_sip_trunk_id` and `livekit_sip_uri`. Vogent is missing from the user docs entirely, even though it has both an integration page and a valid type in the schema.

**Consequence:** A developer building a Vapi or Retell integration reads the user docs, sees only `agent` and `webhook`, and has no idea that provider-specific types exist or what fields they require. They either guess or dig through the API reference to piece it together themselves.

**The fix:** Rewrite `/voice/endpoints` to list all valid type values, show one working example per provider with the exact required fields, add Vogent, and add a "Valid types" callout near the top of the page.

---

### 4. Buy-numbers user docs diverge from the API reference (critical)

**Location:** `/numbers/buy-numbers` vs `/api-reference/reference/phone-numbers`

**Problem:** The user-facing Buy Numbers page and the API reference diverge on field naming, defaults, and response shape for the same `POST /phone-numbers` operation. The API reference uses camelCase request fields (`endpointId`, `totalCostCents`, `setupCostCents`, `monthlyCostCents`) alongside snake_case (`allow_inbound`, `allow_outbound`), documents `allow_outbound` default as `false`, and returns an `e164` field. The reference also describes a "Dashboard flow" requiring `e164`/`country`/`type` and an "API flow" requiring `inventory_id`, but neither set is marked Required in the schema — only prose says "at least one flow's required fields must be present."

**Consequence:** A developer expecting outbound to be enabled by default buys a number and then can't make outbound calls — and the error code returned isn't in the Error Codes reference (see issue 9). An agent parsing the response for `body.number` instead of `body.e164` gets `undefined`. Schema validators can't enforce the oneOf constraint because it's only stated in prose.

**The fix:** Standardize naming conventions across the request body, align the default for `allow_outbound` with what the user docs imply, expose the "at least one flow" constraint in OpenAPI via `oneOf` so validators catch it, and ensure response field names match between the user docs and the schema.

---

### 5. Five default GitBook tutorial pages live in production (critical)

**Location:** `/basics/editor`, `/basics/markdown`, `/basics/images-and-media`, `/basics/interactive-blocks`, `/basics/integrations`

**Problem:** All five pages ship the unmodified GitBook onboarding tutorial — explaining how to use GitBook's block editor, markdown, and integrations — instead of any KrosAI content. They appear in the public `llms.txt` index and resolve at their direct URLs. Example from `/basics/editor`: "GitBook has a powerful block-based editor that allows you to seamlessly create, update, and enhance your content."

**Consequence:** Agents indexing `llms.txt` ingest five pages of vendor product docs that have nothing to do with KrosAI, polluting any RAG store and producing nonsense answers ("To make a call, press `/` to open the insert block menu"). Human readers who land on these pages from search lose trust in the entire site.

**The fix:** Delete the `/basics/*` routes, remove them from `llms.txt`, and configure the GitBook export to skip the default tutorial space.

---

### 6. Help Center is a placeholder shell (critical)

**Location:** `/help-center`

**Problem:** The Help Center page is six category headings ("Getting started," "Integrations," "Make your first call," "Insights & Updates," "Use Cases," "Pricing") with one-line labels and no body content or article links. "Pricing", "Use Cases", "Insights & Updates" point to the marketing site root rather than help articles. The page is reachable from the docs nav and listed in `llms.txt`.

**Consequence:** A customer hitting the Help Center from the docs gets nowhere — there are no articles to read, no FAQ, no troubleshooting paths. Agents indexing the docs ingest six labels and zero content, and will hallucinate that "help articles exist" when nothing is published.

**The fix:** Either populate each category with real articles, or remove the Help Center from the nav and `llms.txt` until it has content. A "Contact support" button alone is not a help center.

---

### 7. Internal scaffolding exposed at `/developer-docs` (significant)

**Location:** `/developer-docs`, `/developer-docs/readme`, `/developer-docs/reference/openapi`

**Problem:** `/developer-docs/readme` ships an internal authoring README: "Production-ready documentation for docs.krosai.com" with the directory layout (`mint.json`, `*.mdx` filenames) and deployment instructions ("Connect this folder to GitBook or Mintlify"). `/developer-docs` itself renders empty ("On this page: developer-docs"). All three are in `llms.txt`.

**Consequence:** Confidential build/deployment internals are publicly indexable, and agents will surface them as "official documentation." It also reveals that the team uses both Mintlify and GitBook tooling, hinting at a half-finished migration.

**The fix:** Move repo-internal README content into the GitHub repo, remove `/developer-docs*` from public navigation and `llms.txt`, and add a 404 or redirect at those URLs.

---

### 8. Two parallel "getting started" trees still indexed (significant)

**Location:** `llms.txt`, `/getting-started-old/quickstart`, `/getting-started-old/quickstart-1`, `/getting-started/*`

**Problem:** The `llms.txt` index lists *both* a current `getting-started/` tree and a legacy `getting-started-old/` tree, with the old tree appearing first. The new quickstart uses `x-api-key` but has a broken step sequence — Step 1, Step 2, Step 3, then an unnumbered "Connect AI Agent Providers" section with an empty body, then Step 4, Step 5, Step 6.

**Consequence:** Agents will preferentially index the older tree because it's listed first under "Documentation" in `llms.txt`, then surface stale examples to users. Humans Googling "krosai quickstart" can land on either page with no canonical signal, and the active quickstart has a hole where an instruction step is supposed to be.

**The fix:** Delete `/getting-started-old/*`, set 301 redirects to the new pages, remove from `llms.txt`, and fix the broken step numbering by either filling the empty section or rolling it into Step 4.

---

### 9. Outbound-calls request schema diverges from user-facing docs (significant)

**Location:** `/voice/outbound-calls` vs `/api-reference/reference/outbound-calls`

**Problem:** The user-facing Outbound Calls page describes fields and behavior that don't match the OpenAPI body schema, which declares only `from_number`, `to_number`, `endpoint_id` ("Optional endpoint to connect after answer"), and `metadata`. The user docs treat `endpoint_id` as required; the spec says optional. The user docs imply `201 Created`; the OpenAPI returns `200`.

**Consequence:** A developer reading the user-facing page and the API reference comes away with different ideas about what fields exist, whether `endpoint_id` is required, and what status code to expect. Calls placed without `endpoint_id` succeed at the API layer per spec but the user docs never explain what happens to that audio.

**The fix:** Reconcile the user-facing Outbound Calls page with the OpenAPI schema — either add the missing fields to the schema if they're real (with defaults), or remove them from the user docs if they aren't. Document the "no endpoint" case explicitly and align the response status code.

---

### 10. Error Codes reference is missing most of the error codes the docs reference (significant)

**Location:** `/reference/error-codes`

**Problem:** The Error Codes page lists 9 codes total (3 auth, 3 numbers, 3 calls). Other pages reference codes that aren't there: `KYC_NOT_SUBMITTED`, `KYC_INITIATED`, `KYC_DECLINED`, `KYC_PENDING`, `MISSING_FIELDS`, `NUMBER_NOT_OWNED`, `OUTBOUND_DISABLED`, `ENDPOINT_NOT_FOUND`, `CONCURRENT_LIMIT_EXCEEDED`, `RATE_LIMIT_EXCEEDED`, `IP_NOT_ALLOWED`, `KEY_EXPIRED`. `500` is listed under HTTP status with no example or remediation.

**Consequence:** When a developer hits `KYC_INITIATED` from `POST /phone-numbers`, they search the canonical reference, find nothing, and have to grep individual endpoint pages. Agents can't produce a single lookup table to map error codes to remediation.

**The fix:** Generate the Error Codes page from the union of all 4xx code lists in the OpenAPI; group by category; add a remediation column for every code including `500`.

---

### 11. `RATE_LIMIT_EXCEEDED` has two contradictory definitions (significant)

**Location:** `/voice/outbound-calls`, `/reference/rate-limits` (referenced)

**Problem:** `/voice/outbound-calls` describes `RATE_LIMIT_EXCEEDED` as "Too many concurrent calls" — a concurrency limit. Elsewhere the same code is used as a request-rate limiter (per-second / per-day throughput in `/reference/rate-limits`). The Error Codes page doesn't reconcile this and the rate-limits page uses a separate code (`CONCURRENT_LIMIT_EXCEEDED`) for concurrency in some places.

**Consequence:** A developer who receives `429 RATE_LIMIT_EXCEEDED` has no deterministic way to know whether to back off on request rate or to close existing calls. The same error code requires opposite remediations in different docs.

**The fix:** Split the code into two — `RATE_LIMIT_EXCEEDED` for request-rate, `CONCURRENT_LIMIT_EXCEEDED` for concurrency — and use them consistently everywhere. Document both in the Error Codes reference with the correct remediation each.

---

### 12. `file:///` local-filesystem links shipped in production (significant)

**Location:** `/numbers/port-numbers` (and any other pages with similar "Next Steps")

**Problem:** The Port Numbers page renders Next Steps as `[View Country Coverage](file:///numbers/countries)`, `[Configure Endpoints](file:///voice/endpoints)`, `[Set Up Webhooks](file:///webhooks/overview)`. These are `file://` URLs — clicking them tries to open the user's local filesystem and fails.

**Consequence:** Every cross-link from the porting flow is dead. A user successfully porting a number is dropped onto a page whose "next" links lead nowhere. Crawlers and agents treat these as broken internal links and downrank the section.

**The fix:** Rewrite the broken links as relative paths (`/numbers/countries`) and audit the entire site for `file:///` artifacts left over from a local Markdown export.

---

### 13. ApiKey scopes documented in Authentication don't match the OpenAPI enum (significant)

**Location:** `/getting-started/authentication` vs `/api-reference/reference/webhooks` (ApiKeyScope schema)

**Problem:** The Authentication page lists scopes including `api-keys:read`, `api-keys:write`, and `voice:connect`. The OpenAPI `ApiKeyScope` enum is `["calls:read","calls:write","numbers:read","numbers:write","endpoints:read","endpoints:write","webhooks:read","webhooks:write","analytics:read","billing:read","billing:write"]`. None of the three scopes from the docs exist; `analytics:read` exists in the API but is omitted from the docs.

**Consequence:** A developer who tries to create a key with `api-keys:write` or `voice:connect` won't find those options in the dashboard. An agent generating API key creation code will request invalid scopes and get a 400.

**The fix:** Pull the scope list directly from the OpenAPI enum, remove the three non-existent scopes, add `analytics:read`, and explain each scope with the endpoints it gates.

---

### 14. Coverage roadmap dates are all in the past (significant)

**Location:** `/numbers/countries`

**Problem:** The "Coming Soon" table lists Morocco/Tunisia at "Q1 2025," Senegal/Ivory Coast/Tanzania/Uganda/Saudi Arabia at "Q2 2025," and Qatar at "Q3 2025." Today is May 2026 — every single date is 1–4 quarters in the past. The changelog has no entries confirming any of these launched.

**Consequence:** Prospective customers evaluating KrosAI for African expansion can't tell which markets actually shipped vs. which slipped. Sales credibility takes a direct hit.

**The fix:** Replace fixed quarters with either current "Available"/"In development" statuses or year-only ETAs ("2026"), and back the table with changelog entries when each market actually launches.

---

### 15. Recording retention contradicts itself and references undocumented endpoints (significant)

**Location:** `/voice/recordings`, `/voice/call-logs` (referenced)

**Problem:** `/voice/recordings` shows a four-tier retention table (Free 7d, Pro 30d, Business 90d, Enterprise custom). `call-logs.mdx` states "Retention: 90 days (configurable)" with no plan qualifier. The page also documents endpoints `POST /calls/{id}/recording/extend` and `PATCH /settings` that don't appear in the OpenAPI paths section, and a per-call `recording_enabled` flag that isn't in the outbound request schema.

**Consequence:** A Free-tier user reads "90 days" in call-logs, discovers after 7 days that recordings were purged, and has no audit trail. Calls to the undocumented extend/settings endpoints either 404 or are an undocumented private API.

**The fix:** Reconcile retention to a single canonical table; if extend/settings endpoints exist, add them to the OpenAPI; if they don't, remove the code samples.

---

### 16. Changelog is empty while OpenAPI claims version 1.3.0 (significant)

**Location:** `/changelog`, `/developer-docs/reference/openapi`

**Problem:** The changelog page says: "We haven't published any changes at this time" and then lists only "v1.0.0 (Current)" with the initial release notes. Meanwhile every OpenAPI block in the API reference reports `"version":"1.3.0"`, and the openapi spec page lists "v1.0.0 (Current)". Three minor versions of API changes are entirely undocumented.

**Consequence:** Customers running production integrations have no way to learn about behavior changes between 1.0.0 → 1.3.0. There is no deprecation signal, no breaking-change notice, no field-add notice. Agents will tell users they're on 1.0.0 because that's what `/developer-docs/reference/openapi` says.

**The fix:** Backfill changelog entries from git history for 1.1.0, 1.2.0, 1.3.0, update the openapi page version, and wire the API's runtime version into the docs build so they can't drift again.

---

### 17. Rate limits page mixes tables and has a blank escalation address (significant)

**Location:** `/reference/rate-limits`

**Problem:** The page has two tables with the same per-plan numbers: a "Limits by Plan" table with a "Concurrent Calls" column (Free 1, Pro 10, Business 50) and a separate "Concurrent Call Limits" section titled "Concurrent Outbound" with the same values. Readers can't tell whether Free's "1" applies to total simultaneous calls (inbound + outbound) or only outbound. The "Requesting Higher Limits" section literally reads `Email  with:` with a blank where the support email address should be.

**Consequence:** A Pro customer running 10 outbound calls doesn't know if an 11th inbound call will be blocked. The blank email address means the documented escalation path is non-functional — there is no way for a customer to "request higher limits" via the documented channel.

**The fix:** Merge the two tables into one with separate Inbound/Outbound/Total columns, and either insert the support email address or link to the dashboard support page so the escalation path actually works.

---

### 18. React SDK Quick Start is empty (significant)

**Location:** `/getting-started/sdks`

**Problem:** The SDKs page advertises `@krosai/voice-sdk` and `@krosai/voice-sdk-react` as official SDKs. The React SDK section renders only labels — "Installation / Quick Start / Hooks" — with no code in the Quick Start body. Python, Go, Ruby, and PHP libraries are flagged "Coming Soon" with the contact email scrubbed.

**Consequence:** A developer who chose KrosAI partly because a React SDK exists arrives at a page that doesn't show them how to use it. There's no install command followed by a working snippet — just headings. Agents asked "how do I use the KrosAI React SDK" can only echo the headings back.

**The fix:** Populate the React SDK Quick Start with an installation command, a minimal `<KrosaiProvider>`-style example, and the most common hook (e.g., `useCall`). If the SDK is not yet ready for public use, move it to "Coming Soon" alongside Python.

---

### 19. Vapi integration guide has visible typos and copy-paste blocks (minor)

**Location:** `/integration/vapi`, `/integration/retell`, `/integration/elevenlabs`, `/integration/vogent`

**Problem:** Visible typos in step labels on the Vapi page: "Dashobard," "Intergrations," "credientials," "dashoard," and grammar slips like "Click of Import SIP Phone Number" and "click of Endpoints." The "Make your first call" celebration block is duplicated verbatim across multiple provider integration guides. Vapi prose calls `credential_id` required, but the OpenAPI schema declares it Optional.

**Consequence:** Erodes confidence; for a developer following Vapi setup, the `credential_id` contradiction may cause them to fail validation or skip a required step.

**The fix:** Spell-check all integration guides, factor the "first call" block into a shared snippet, and reconcile `credential_id` (required vs optional) between prose and schema.

---

### 20. `fax` capability exists in the schema but nowhere in the narrative docs (minor)

**Location:** `/numbers/countries`, `/api-reference/reference/phone-numbers` (inventory schema)

**Problem:** The OpenAPI phone-number inventory schema exposes a `fax` capability alongside voice/SMS, but no narrative page mentions fax. Country detail tables describe `capabilities.sms` and `capabilities.voice`; `fax` is unmentioned. Either KrosAI sells fax-capable numbers (and the docs hide the feature) or the schema field is dead surface area.

**Consequence:** Either developers can't discover a feature the API supports, or the API is shipping schema fields it doesn't honor. Agents will surface `fax: true` from inventory responses without any documented behavior for what that enables.

**The fix:** If fax is a real capability, document it on `/numbers/countries` and add a section on how fax routing works. If it's not, remove the field from the schema.

---

### 21. LiveKit endpoints have no documented error-state contract (minor)

**Location:** `/integration/livekit`, `/api-reference/reference/endpoints`

**Problem:** The LiveKit integration page warns that "agent verification happens at call-time dispatch" because LiveKit agents are ephemeral workers. The OpenAPI Endpoint schema includes `status: "error"` in its enum, but the LiveKit page never documents what causes an endpoint to enter `error`, how to detect it, how to recover, or whether an error status surfaces as a webhook event.

**Consequence:** Developers integrating LiveKit will learn empirically — through failed calls — what makes an endpoint go into `error` and how to recover. There's no documented troubleshooting flow.

**The fix:** Add a "Failure modes" section to the LiveKit integration page enumerating the conditions that produce `status: "error"`, how to detect transition into and out of `error`, and the corresponding webhook event (if any).

---

### 22. Internal link path style is inconsistent (minor)

**Location:** `/getting-started/introduction` "Quick links" block

**Problem:** The Introduction's Quick Links use mixed link conventions in the same block: `https://docs.krosai.com/api-reference/` (absolute URL with trailing slash, points to an index) alongside `/voice/outbound-calls` (relative path, no trailing slash) and `/webhooks/overview` (relative, no slash). No clear rule for when to use which.

**Consequence:** Minor consistency footgun; agents indexing internal link graphs may treat the absolute and relative variants as different destinations, and contributors won't know which style to use.

**The fix:** Standardize on relative paths without trailing slashes across the docs, and lint for the convention in CI.

---

### 23. OpenAPI title mismatch between docs and embedded spec (minor)

**Location:** `/developer-docs/reference/openapi`

**Problem:** The openapi page describes "KrosAI API" and links to `https://api.krosai.com/v1/openapi.json` and `https://api.krosai.com/docs`. But the embedded OpenAPI title throughout the rendered reference is "KrosAI Cockpit API" — a different name. The page also lists "v1.0.0 (Current)" while the embedded spec reports `"version":"1.3.0"`.

**Consequence:** Readers can't tell whether "Cockpit API" is a separate internal product or just an old name for the same API. Agents indexing the spec metadata will record `KrosAI Cockpit API` as the product name.

**The fix:** Rename the OpenAPI `info.title` to "KrosAI API," align the version on this page with the spec, and confirm `https://api.krosai.com/docs` is publicly accessible (or remove the link).

---

## What the Docs do well

- The OpenAPI spec exists and is reasonably complete - most contradictions are resolvable by treating it as source of truth.
- `llms.txt` is present at a discoverable URL, which already puts KrosAI ahead of many peers for agent indexing.
- Coverage of niche markets (Nigeria, Ghana, Kenya, Rwanda, Egypt, UAE) is a clear differentiator and the country detail pages do list per-country pricing.

## My Top 3 recommendations

1. **Pick one source of truth (the OpenAPI spec) and regenerate every API-shaped table from it** - webhook events, scopes, error codes, endpoint types, request/response field names. The contradictions in issues 1, 2, 3, 4, 9, 13, and 15 collapse into one process fix.
2. **Delete the legacy and accidental pages and populate the empty ones** - `/getting-started-old/*`, `/basics/*`, `/developer-docs/*`, the `file:///` links in `/numbers/port-numbers`, and the empty Help Center and React SDK Quick Start. Remove dead surfaces from `llms.txt`. This stops the docs from poisoning agent indexes immediately.
3. **Backfill the changelog from 1.0.0 → 1.3.0 and wire the live API version into the build** so the docs can't claim 1.0.0 while the API serves 1.3.0. Pair this with a credible "Coming Soon" coverage roadmap that isn't 4 quarters stale, and unblock the documented support-email escalation path on `/reference/rate-limits`.
