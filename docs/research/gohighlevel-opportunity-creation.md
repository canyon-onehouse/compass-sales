# GHL Opportunity creation at lead capture

Research for [issue #21](https://github.com/canyon-onehouse/compass-sales/issues/21). All claims cite the primary source that owns them — GoHighLevel's own help center (`help.gohighlevel.com`), its developer marketplace docs (`marketplace.gohighlevel.com/docs`), and its public OpenAPI source repo ([`github.com/GoHighLevel/highlevel-api-docs`](https://github.com/GoHighLevel/highlevel-api-docs), the repo GHL itself publishes as the source documentation for API v2). Researched 2026-08-07. As with the [lead-capture research](gohighlevel-lead-capture.md), the marketplace docs site is a JS-rendered app that automated fetch only partially extracts, so exact field names and required-field lists are quoted from the raw OpenAPI JSON ([`opportunities.json`](https://raw.githubusercontent.com/GoHighLevel/highlevel-api-docs/main/apps/opportunities.json), [`contacts.json`](https://raw.githubusercontent.com/GoHighLevel/highlevel-api-docs/main/apps/contacts.json)) that powers that site.

The question: when the site's server creates a Contact via `POST /contacts/` (per the [Inquiry destination decision, #11](https://github.com/canyon-onehouse/compass-sales/issues/11)), should it also create the pipeline Opportunity via API — or leave Opportunity creation to a GHL workflow that triggers on the new contact? The user prefers the workflow path; this doc verifies that preference.

---

## Headline answers

| Question | Answer |
| --- | --- |
| Does API v2 expose Opportunity creation? | Yes. `POST /opportunities/` on `services.leadconnectorhq.com`, scope `opportunities.write`, `Version: 2021-07-28` header. Required body fields: `pipelineId`, `locationId`, `name`, `status`, `contactId` ([opportunities.json](https://raw.githubusercontent.com/GoHighLevel/highlevel-api-docs/main/apps/opportunities.json), `CreateDto`). `pipelineStageId` is optional in the schema. |
| Can pipeline/stage IDs be looked up via API? | Yes — `GET /opportunities/pipelines?locationId=…`, scope `opportunities.readonly` ([opportunities.json](https://raw.githubusercontent.com/GoHighLevel/highlevel-api-docs/main/apps/opportunities.json)). The OpenAPI under-specifies each pipeline's `stages` array shape (§1.3). |
| Does a workflow trigger fire for an API-created contact? | Yes — the **Contact Created** trigger fires "regardless of how the contact is created—manually, via form submissions, or through integrations," with bulk imports the one documented exclusion ([help: Workflow Trigger – Contact Created](https://help.gohighlevel.com/support/solutions/articles/155000002486-workflow-trigger-contact-created)). It supports tag and custom-field filters. |
| Can a workflow create the Opportunity, with pipeline + stage? | Yes — the **Create Opportunity** action sets pipeline, stage, name, source, status, value, owner, and opportunity custom fields, and has a per-action duplicate toggle keyed on contact ID ([help: Workflow Action – Create Opportunity](https://help.gohighlevel.com/support/solutions/articles/155000004752-workflow-action-create-opportunity)). |
| Does `POST /contacts/` upsert on duplicate email/phone? | Not documented to. The duplicate-aware endpoint is **`POST /contacts/upsert`**, which explicitly follows the location's "Allow Duplicate Contact" setting and returns a `new` boolean ([contacts.json](https://raw.githubusercontent.com/GoHighLevel/highlevel-api-docs/main/apps/contacts.json)). Plain `POST /contacts/` documents `400`/`422` errors but not their duplicate semantics (§3.1 — gap). |
| Is the "handle it inside GHL" preference well-founded? | Yes, with one caveat: **Contact Created does not re-fire for a recognized existing contact**, so a returning lead's second inquiry creates no second Opportunity (and no workflow run at all) unless the trigger is tag- or field-based (§4). |

---

## 1. The Opportunities API v2

### 1.1 Create Opportunity

Confirmed directly from [opportunities.json](https://raw.githubusercontent.com/GoHighLevel/highlevel-api-docs/main/apps/opportunities.json):

- **`POST /opportunities/`**, summary "Create Opportunity", scope **`opportunities.write`**, required header `Version: 2021-07-28` (single enum value), `201` on success (`400`/`401`/`422` documented as errors, bodies from the shared `common-schemas.json`).
- **Required body fields** (`CreateDto.required`): `pipelineId`, `locationId`, `name`, `status`, `contactId`. `status` is an enum: `open`, `won`, `lost`, `abandoned`, `all`.
- **Optional fields**: `pipelineStageId` (example value is a UUID), `monetaryValue`, `assignedTo`, and `customFields` (string/array/object input schemas — Opportunity-level custom fields, distinct from Contact-level ones per the [custom-fields modeling rule](gohighlevel-lead-capture.md) already on record).
- Notably, `pipelineStageId` is **not** required by the schema. The workflow action's documented behavior — "defaults to first stage if unspecified" ([help: Create Opportunity action](https://help.gohighlevel.com/support/solutions/articles/155000004752-workflow-action-create-opportunity)) — is not stated for the API endpoint; what the API does when the stage is omitted is undocumented (§5).

**Auth:** the spec's security scheme is the same one already verified for Contacts: *"Use the Access Token generated with user type as Sub-Account (OR) Private Integration Token of Sub-Account"* ([opportunities.json](https://raw.githubusercontent.com/GoHighLevel/highlevel-api-docs/main/apps/opportunities.json), `components.securitySchemes.bearer`). So the existing Private Integration Token approach works — the token just needs the opportunities scopes granted at creation time. The Private Integrations help article confirms scopes are chosen per-token ("Select the scopes/permissions that you want the private integration to have access to") but does not enumerate the scope list; it also documents 90-day rotation guidance, the 7-day dual-token grace window, and a limit of 5 tokens per level ([help: Private Integrations – Everything You Need to Know](https://help.gohighlevel.com/support/solutions/articles/155000003054-private-integrations-everything-you-need-to-know)).

### 1.2 Upsert Opportunity — exists, but oddly shaped

**`POST /opportunities/upsert`** exists (scope `opportunities.write`, `200` response whose body includes the opportunity plus a **`new` boolean**). But its `UpsertOpportunityDto` is surprising: required fields are `pipelineId`, `locationId`, `followers`, `isRemoveAllFollowers`, `followersActionType` — and the schema has **no `contactId` property at all**. The operation description is just "Upsert Opportunity"; **what key it matches on (opportunity `id`? something else?) is undocumented** ([opportunities.json](https://raw.githubusercontent.com/GoHighLevel/highlevel-api-docs/main/apps/opportunities.json)). Do not build on this endpoint without testing it against the live account (§5).

Other write operations for completeness: `PUT /opportunities/{id}` (Update), `PUT /opportunities/{id}/status`, `DELETE /opportunities/{id}`, follower add/remove — all `opportunities.write`. Read side: `GET /opportunities/search`, `GET /opportunities/{id}`, `GET /opportunities/lost-reason` — all `opportunities.readonly`.

### 1.3 Looking up pipeline and stage IDs

- **`GET /opportunities/pipelines`** with required query param `locationId`, scope `opportunities.readonly`, returns `{ pipelines: [ { id, name, stages, showInFunnel, showInPieChart, locationId, colorRenderMode } ] }` ([opportunities.json](https://raw.githubusercontent.com/GoHighLevel/highlevel-api-docs/main/apps/opportunities.json), `PipelinesResponseSchema`).
- The spec declares `stages` as an array whose items are typed only as `array` — the stage object's shape (id/name fields) is **not specified** in the OpenAPI. Given that `pipelineStageId` examples are UUIDs, stages almost certainly carry `{id, name}`, but the spec doesn't say so (§5).
- On the UI side, GHL's own help admits there is no in-app ID display: *"If you do not wish to use Zapier for Stage ID and Pipeline ID, in HighLevel the only way to get that information is by exporting opportunities"* ([help: Finding the Pipeline and Stage ID using Zapier](https://help.gohighlevel.com/support/solutions/articles/48001160284-finding-the-pipeline-and-stage-id-using-zapier)). For this project the API lookup above is the sane path — but note this is a one-time (or occasional) lookup whose results would then live in site config; that is exactly the coupling the workflow path avoids (§4).

---

## 2. The workflow path

### 2.1 Trigger: Contact Created

The **Contact Created** workflow trigger fires "when a new contact is added to the system" and — the load-bearing sentence for this ticket — "works regardless of how the contact is created—manually, via form submissions, or through integrations" ([help: Workflow Trigger – Contact Created](https://help.gohighlevel.com/support/solutions/articles/155000002486-workflow-trigger-contact-created)). The article does not name the public API explicitly, but an API-created contact falls under contact creation generally; the article's only documented exclusion is bulk import: "Bulk imports do not trigger this workflow to prevent accidental automation overload."

Two more facts from the same article that matter here:

- **Filters:** the trigger supports filters on **tags** (Equals / Not equals / Any of / None of, combinable with AND logic) and on **custom fields**. Since the site's `POST /contacts/` payload can set both `tags` and `customFields` ([contacts.json](https://raw.githubusercontent.com/GoHighLevel/highlevel-api-docs/main/apps/contacts.json), `CreateContactDto`), the site can mark website leads (e.g. tag `website-inquiry`) and the workflow can fire only for them — keeping manual contact entry from spawning Opportunities.
- **No re-fire on recognized duplicates:** if "a contact with the same email or phone number is added again" and the system recognizes it as existing, "the workflow will not trigger again." The article explicitly notes behavior depends on the location's duplicate-detection settings. Consequence analyzed in §3.3/§4.

### 2.2 Trigger alternative: Contact Tag

The **Contact Tag** trigger fires "whenever a contact tag is added or removed," configurable per-tag and per-direction, and fires "after the tag update is actually applied" ([help: Workflow Trigger – Contact Tag](https://help.gohighlevel.com/support/solutions/articles/155000002482-workflow-trigger-contact-tag)). The article does not state whether adding an *already-present* tag re-fires the trigger, nor does it explicitly address tags applied via API (§5). This trigger is the natural escape hatch if re-submissions by existing contacts need to produce workflow runs (§3.3) — pending a live test of the already-present-tag case.

### 2.3 Action: Create Opportunity

The current-generation **Create Opportunity** workflow action ([help: Workflow Action – Create Opportunity](https://help.gohighlevel.com/support/solutions/articles/155000004752-workflow-action-create-opportunity)) configures:

- **Pipeline and stage** — chosen in the workflow UI; stage "defaults to first stage if unspecified."
- **Opportunity name** — defaults to the contact's name; supports dynamic values like `{{contact.first_name}}`.
- **Source, status** (defaults to Open), **value** (defaults to 0), **owner** (may auto-assign to the contact's owner), and **custom fields**.
- **Per-action duplicate toggle:** "Enables or disables the creation of a new opportunity if one already exists for the same contact ID," with the article stressing "Duplicate logic is based on contact ID, not the opportunity name."
- The action "cannot create opportunities without an associated contact" — fine here, since the triggering contact is the association.

The older combined **Create/Update Opportunity** action still exists but "is being phased out. It is recommended to use Create Opportunity or Update Opportunity actions instead" — the phase-out applies "only for new workflows" ([help: Workflow Action – Create/Update Opportunity](https://help.gohighlevel.com/support/solutions/articles/155000002476-workflow-action-create-update-opportunity)). That legacy action carried an "Allow Duplicate Opportunities" checkbox; the new action's contact-ID-keyed toggle (above) is its replacement. A separate **Update Opportunity** action exists for later-stage automation ([help: Workflow Action – Update Opportunity](https://help.gohighlevel.com/support/solutions/articles/155000004753-workflow-action-update-opportunity)).

### 2.4 The location-level "Allow Duplicate Opportunity" setting

Distinct from the per-action toggle, there is a location-level setting: *"Turning this feature 'ON' will allow duplicate opportunities to be created in the same pipeline. If this is turned 'ON', new opportunities will be created in the same pipeline instead of an opportunity moving from one pipeline stage to another."* It now lives under **Objects → Opportunities → Details** ([help: Business Profile Settings – General](https://help.gohighlevel.com/support/solutions/articles/155000006221-business-profile-settings-general)). The Opportunities FAQ adds a related constraint: "If you do not have 'Allow duplicate opportunities' setting on, one contact cannot be added to two different opportunity's primary/additional contacts list" ([help: Opportunities FAQs](https://help.gohighlevel.com/support/solutions/articles/155000002000-opportunities-faqs)). Whether `POST /opportunities/` honors this setting (rejects? silently dedupes?) is not documented anywhere found (§5).

---

## 3. Duplicate behavior on re-submission

### 3.1 `POST /contacts/` vs `POST /contacts/upsert`

- **Plain `POST /contacts/` (Create Contact)** documents `201`/`400`/`401`/`422` but its description says nothing about duplicate matching — the OpenAPI does **not** state whether an existing email/phone yields an error or a silent update ([contacts.json](https://raw.githubusercontent.com/GoHighLevel/highlevel-api-docs/main/apps/contacts.json)). No help-center or marketplace-docs page found documents plain Create's duplicate semantics (only third-party SEO content discusses the observed 400, which this doc does not treat as a source). **Gap carried forward** (§5) — but a moot one if the site uses upsert, which it should.
- **`POST /contacts/upsert` (Upsert Contact)** — note the method: **POST**, correcting the "PUT" slip in the earlier [lead-capture doc](gohighlevel-lead-capture.md) §3.1 — is explicitly documented as duplicate-aware: *"The Upsert API will adhere to the configuration defined under the 'Allow Duplicate Contact' setting at the Location level. If the setting is configured to check both Email and Phone, the API will attempt to identify an existing contact based on the priority sequence specified in the setting, and will create or update the contact accordingly."* If two different existing contacts match email and phone respectively, it updates the one matching the first field in the configured sequence ([contacts.json](https://raw.githubusercontent.com/GoHighLevel/highlevel-api-docs/main/apps/contacts.json), operation description). The response body includes a **`new` boolean** (`UpsertContactsSuccessfulResponseDto`), so the site can tell created-vs-updated apart. An optional `createNewIfDuplicateAllowed` flag forces immediate creation *only* when the location allows duplicate contacts; otherwise normal upsert behavior applies (same source, `UpsertContactDto`).
- The **"Allow Duplicate Contact"** setting lives at Settings → Business Profile → Contact Deduplication Preferences, with a primary match field (Email or Phone) and optional secondary ([help: Allow Duplicate Contact Explained](https://help.gohighlevel.com/support/solutions/articles/48001181714-allow-duplicate-contact-explained)). There is also `GET /contacts/search/duplicate` (scope `contacts.readonly`) for an explicit pre-check by email/phone ([contacts.json](https://raw.githubusercontent.com/GoHighLevel/highlevel-api-docs/main/apps/contacts.json)).

### 3.2 What re-submission means on the API path

If the site creates the Opportunity itself, then on every form submission it would call Create/Upsert Contact **and** `POST /opportunities/`. The Opportunity create endpoint has no documented dedup semantics (§2.4 note, §5), so a returning contact re-submitting the form yields a second Opportunity — or the site must implement its own guard (search opportunities by `contactId` via `GET /opportunities/search`, then decide). That is real logic, error handling, and API-quota surface living in site code.

### 3.3 What re-submission means on the workflow path

With a Contact-Created-triggered workflow: the first submission upserts a new contact (`new: true`), the trigger fires, the Opportunity is created. A later re-submission by the same person upserts the *existing* contact (`new: false`) — and per §2.1, **Contact Created will not fire again**, so no duplicate Opportunity. The flip side: the returning lead's second inquiry produces **no workflow run at all** — no notification, no re-opened Opportunity. If that matters, the documented lever is a tag-based trigger (§2.2): the site's upsert payload always includes the `website-inquiry` tag, and a Contact Tag workflow fires on tag-add — with the open question of whether re-adding an already-present tag fires it (§5). The Create Opportunity action's own duplicate toggle (contact-ID-keyed, §2.3) then decides whether a second Opportunity is allowed.

---

## 4. Trade-offs for a sales team of one — verdict

The user's preference for handling Opportunity creation inside GHL is **well-founded**. Weighing the two paths:

| Concern | API path (site creates Opportunity) | Workflow path (GHL creates Opportunity) |
| --- | --- | --- |
| IDs in site code | `pipelineId` (+ `pipelineStageId`) must live in site config, fetched once via `GET /opportunities/pipelines` and hardcoded thereafter | None — pipeline and stage are picked from dropdowns in the workflow builder ([Create Opportunity action](https://help.gohighlevel.com/support/solutions/articles/155000004752-workflow-action-create-opportunity)); the site only ever sends the contact |
| Token scopes | PIT needs `contacts.write` **and** `opportunities.write` | `contacts.write` only — smaller blast radius, in line with GHL's own "grant as few scopes as necessary" guidance ([Private Integrations](https://help.gohighlevel.com/support/solutions/articles/155000003054-private-integrations-everything-you-need-to-know)) |
| Pipeline restructures | Stage changes mean a config/code change and redeploy | Edited in the same GHL UI where the restructure happens |
| Duplicate handling | Undocumented for `POST /opportunities/`; site must build its own guard (§3.2) | Documented at two levels: per-action contact-ID toggle + location-level Allow Duplicate Opportunity setting (§2.3–2.4) |
| Failure surface | Two sequential API calls per lead; partial-failure handling (contact created, opportunity failed) is site code | One API call per lead; the Opportunity step runs inside GHL where its execution history is visible in the workflow |
| Later evolution | More code for each new step (assign owner, notify, tag) | Same workflow gains actions without touching the site |

One honest caveat in the other direction: renames alone don't obviously break the API path either — IDs are the reference, and nothing found documents IDs changing on rename — but nothing documents them *not* changing, and stage *deletions/recreations* certainly would. The workflow path removes the whole class of question.

**Recommended shape** (consistent with #11's decision): site → `POST /contacts/upsert` with a distinguishing tag → GHL workflow (Contact Created, tag-filtered — or Contact Tag if re-submission runs prove necessary and the re-add behavior tests out) → Create Opportunity action with the duplicate toggle set to the studio's taste. Grant the PIT `contacts.write`; add `opportunities.readonly`/`opportunities.write` only if a concrete need appears later.

---

## 5. Gaps carried forward

Things the primary sources do **not** establish, to be resolved by testing against the live account or future docs:

1. **Plain `POST /contacts/` duplicate semantics** — the OpenAPI lists `400`/`422` without saying what happens on an existing email/phone. Moot if the site standardizes on `POST /contacts/upsert`, which it should.
2. **Whether `POST /opportunities/` honors "Allow Duplicate Opportunity"** — no doc found states whether the API rejects, dedupes, or ignores the location setting; likewise what the API does when `pipelineStageId` is omitted (the first-stage default is documented only for the workflow *action*).
3. **`POST /opportunities/upsert` matching semantics** — the DTO has no `contactId` and requires follower fields; what it upserts *on* is undocumented. Avoid until tested.
4. **The `stages` shape in `GET /opportunities/pipelines`** — the OpenAPI types stage items only as `array`; the presumed `{id, name}` shape needs one live call to confirm (only matters if the API path is ever revisited).
5. **Contact Tag trigger on re-added tags and API-applied tags** — the help article doesn't state whether adding an already-present tag fires the trigger, nor explicitly that API-applied tags do. Load-bearing only if returning-lead re-submissions must produce workflow runs — test before relying on it.
6. **"Contact Created" and the public API, verbatim** — the trigger article says creation channel doesn't matter and excludes only bulk imports, but never names the REST API explicitly. Near-certain, cheap to verify with one test submission once the workflow exists.
7. **Exact scope names in the Private Integrations UI** — the help article confirms per-token scope selection but doesn't enumerate the list; the OpenAPI's `opportunities.readonly`/`opportunities.write` are the API-side names. Confirm the checkbox labels when creating the token.
