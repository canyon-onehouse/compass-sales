# GoHighLevel lead-capture capabilities

Research for [issue #8](https://github.com/canyon-onehouse/compass-sales/issues/8). All claims cite the primary source that owns them — GoHighLevel's own help center (`help.gohighlevel.com`), its developer marketplace docs (`marketplace.gohighlevel.com/docs`), and its public OpenAPI source repo ([`github.com/GoHighLevel/highlevel-api-docs`](https://github.com/GoHighLevel/highlevel-api-docs), the repo GHL itself publishes as "the source documentation for the GoHighLevel API V2"). Researched 2026-08-07. Where `marketplace.gohighlevel.com/docs` pages are noted, they are a JS-rendered API-reference app; several were only partially extractable by automated fetch, so this doc cross-checks their claims against the raw OpenAPI JSON in the GitHub repo (which powers that same site) wherever possible — that JSON is quoted directly and is the more load-bearing source for exact field names.

This research does not make the Inquiry-form / Plan-request-form design decision. It lays out what GHL can and can't do natively so that decision can be made.

---

## Headline answers

| Question | Answer |
| --- | --- |
| Can a Next.js site just embed two GHL-hosted forms? | Yes — GHL forms are built in-app and produce a copy-pasteable "embed code" for any external site ([help.gohighlevel.com/.../embedding-highlevel-forms-on-non-highlevel-websites](https://help.gohighlevel.com/support/solutions/articles/155000004524-embedding-highlevel-forms-on-non-highlevel-websites)). GHL's own docs don't discuss iframe height/CSP/hydration behavior in a JS-framework context (see §2.3). |
| Can the Next.js site POST leads directly to GHL's API instead of embedding a form? | Yes — `POST /contacts/` on the public API v2, auth'd with a Bearer token (OAuth access token or a Private Integration Token), `locationId` is the only required field ([contacts.json](https://raw.githubusercontent.com/GoHighLevel/highlevel-api-docs/main/apps/contacts.json), confirmed against `CreateContactDto`). There is **no public "submit a form" write endpoint** — the Forms API v2 is read-only (`GET /forms/`, `GET /forms/submissions`) ([forms.json](https://raw.githubusercontent.com/GoHighLevel/highlevel-api-docs/main/apps/forms.json)). |
| Can a custom field capture "which plan"? | Yes. Contact-level custom fields are defined once (Settings → Custom Fields, or `POST /custom-fields/`) and then set per-contact either through a form field or via the `customFields` array (`{id, key, field_value}`) on Create/Update Contact ([custom-fields.json](https://raw.githubusercontent.com/GoHighLevel/highlevel-api-docs/main/apps/custom-fields.json); [contacts.json](https://raw.githubusercontent.com/GoHighLevel/highlevel-api-docs/main/apps/contacts.json)). |
| Can source be distinguished per-form / per-plan without custom API work? | Partially. Which *form* a submission came from is tracked natively (workflows can filter by exact form name) ([help: Workflow Trigger – Form Submitted](https://help.gohighlevel.com/support/solutions/articles/155000002550-workflow-trigger-form-submitted)), and a hidden `source` field can be overridden per link via `?source=...` ([help: Source Field in Forms and Surveys](https://help.gohighlevel.com/support/solutions/articles/155000001506-source-field-in-forms-and-surveys)). Passing a *dynamic* value like "which plan" into a form field via URL requires the incoming URL parameter's key to exactly match the field's configured key ([help: URL Parameters in Websites and Funnels](https://help.gohighlevel.com/support/solutions/articles/155000005722-url-parameters-in-websites-and-funnels)) — GHL's own feature-request board shows a related ask ("prefill via URL params") still under review, not yet a documented, guaranteed capability ([ideas.gohighlevel.com](https://ideas.gohighlevel.com/forms/p/url-parameters-to-fill-custom-fields-in-forms)). |
| Any official Next.js/React integration guide? | None found. No page on `developers.gohighlevel.com`, `marketplace.gohighlevel.com/docs`, or `help.gohighlevel.com` mentions Next.js, React, SSR, or hydration. |

---

## 1. Hosted forms

### 1.1 What it is

GHL's native form builder lives inside a sub-account under **Sites → Forms** ([help: How to Create a Contact Form in HighLevel](https://help.gohighlevel.com/support/solutions/articles/155000004549-how-to-create-a-contact-form-in-highlevel-)). You click "+ Create New Form," drag-and-drop fields (Full Name, Email, Phone, Message, plus any contact-level custom field), and save. Every saved form gets:

- A **hosted URL** of the form `https://link.gohighlevel.com/widget/form/[ID]` — this is the URL that source/UTM overrides get appended to ([help: Source Field in Forms and Surveys](https://help.gohighlevel.com/support/solutions/articles/155000001506-source-field-in-forms-and-surveys)).
- An **embed code**, generated from the form's "Integrate" tab ([help: Embedding HighLevel Forms on Non-HighLevel Websites](https://help.gohighlevel.com/support/solutions/articles/155000004524-embedding-highlevel-forms-on-non-highlevel-websites)).

### 1.2 Customization available

- **Fields:** standard fields plus any contact-level custom field; dropdowns, checkboxes, radio buttons; drag-to-reorder; per-field "Required" toggle ([help: "Add Contact" Form Upgrade and Customizations](https://help.gohighlevel.com/support/solutions/articles/155000006513--add-contact-form-upgrade-and-customizations); [help: How to Create a Contact Form in HighLevel](https://help.gohighlevel.com/support/solutions/articles/155000004549-how-to-create-a-contact-form-in-highlevel-)).
- **Style:** background, text color, spacing, a "Full Width" responsive toggle ([help: How to Create a Contact Form in HighLevel](https://help.gohighlevel.com/support/solutions/articles/155000004549-how-to-create-a-contact-form-in-highlevel-)).
- **Post-submit behavior:** show a custom thank-you message, redirect to a URL, or trigger a workflow ([help: How to Create a Contact Form in HighLevel](https://help.gohighlevel.com/support/solutions/articles/155000004549-how-to-create-a-contact-form-in-highlevel-)).
- **Notifications:** email notifications to staff and/or the submitter, configured per form.

### 1.3 Embed presentation options

The dedicated embedding-options article ([help: How to Use Embedding Options for Forms](https://help.gohighlevel.com/support/solutions/articles/155000004538-how-to-use-embedding-options-for-forms-triggers-layouts-and-deactivation-settings-explained)) documents four layouts an embedded form can render as:

- **Inline** — embedded directly as a page element (no popup behavior; the layout relevant to a normal marketing-site form).
- **Popup** — modal overlay.
- **Polite Slide-In** — slides in from a screen edge.
- **Sticky Sidebar** — persists at a screen edge while scrolling.

Non-inline layouts get a configurable modal height (default 500 px, any whole number ≥ 1); this setting doesn't apply to Inline. Trigger conditions (scroll %, time elapsed, always-on) and activation/deactivation rules (show after N visits, stop after N displays or after a lead submits) are also configured here. The article explicitly warns: **"Embed codes are static once they are placed on a website"** — changing form config after embedding requires regenerating and re-pasting the code.

---

## 2. Embeddable forms

### 2.1 How embedding works

The embed flow is: open the form → **Integrate** tab → **Copy Embed Code** → paste into the target site ([help: Embedding HighLevel Forms on Non-HighLevel Websites](https://help.gohighlevel.com/support/solutions/articles/155000004524-embedding-highlevel-forms-on-non-highlevel-websites)). The article gives platform-specific paste locations for Squarespace, Wix, Shopify, WordPress, and Duda, phrased generically as "embed code"/"embed script" — **it does not state in its rendered text whether the underlying mechanism is an `<iframe>` tag or a JS-injected widget**, and it doesn't publish the code snippet's literal markup on the page. It does call out one platform constraint: "Squarespace does not support embedding of forms on checkout pages."

### 2.2 Not covered by this help article

The embedding article is silent on: iframe auto-resizing/height behavior, Content-Security-Policy requirements, and script-loading strategy (defer/async) — none of these terms appear on the page.

### 2.3 Next.js-specific implications (inferred, not GHL-documented)

No GHL page mentions Next.js, React, SSR, or hydration. Practical implications a Next.js integration would need to work out without vendor guidance:
- If the embed is a `<script>` tag that injects DOM (common for widget-style embeds like the four layouts in §1.3), it would need to run as a client component and be reconciled with React's ownership of the DOM subtree it's mounted into — a pattern Next.js's own docs describe generically for third-party scripts (not GHL-specific, so not cited here as a GHL fact).
- If it's a literal `<iframe src="https://link.gohighlevel.com/widget/form/...">`, standard iframe considerations apply (no built-in auto-resize is documented by GHL; CSP `frame-src`/`frame-ancestors` allowances would need to include GHL's embed domain, which the fetched pages don't name explicitly beyond the hosted-form URL prefix `link.gohighlevel.com`).
- This is flagged as a gap in §7 — the actual embed snippet's markup should be pulled from a live GHL account before implementation, since the help article doesn't reproduce it.

### 2.4 A separate, distinct mechanism: "External Tracking" of non-GHL forms

Worth distinguishing because it's easy to conflate: GHL also has an **"External Tracking"** feature (Settings → External Tracking) that is the *opposite* direction — it's a script installed on a site to detect and capture submissions from forms **not built in GHL** (e.g., Gravity Forms, WPForms, Contact Form 7, custom HTML forms), turning them into GHL contacts with attribution ([help: Tracking External Forms with GoHighLevel](https://help.gohighlevel.com/support/solutions/articles/155000006092-tracking-external-forms-with-gohighlevel)). It explicitly **does not support forms embedded in iframes or controlled by third-party widgets** — it requires the form to be a real DOM `<form>` element with visible `name`-attributed inputs. This would matter only if the Inquiry/Plan-request forms end up being built as native Next.js `<form>` elements (not GHL-hosted/embedded) with GHL wired in as a listener rather than the form's origin.

---

## 3. Webhooks / inbound API for lead capture

### 3.1 Direct API: Create Contact

Confirmed directly from the OpenAPI source GHL publishes ([contacts.json](https://raw.githubusercontent.com/GoHighLevel/highlevel-api-docs/main/apps/contacts.json)):

- **`POST /contacts/`** on base URL `https://services.leadconnectorhq.com`.
- **Auth:** `Authorization: Bearer <token>` where the token is either an OAuth access token issued to a Marketplace app, or a **Private Integration Token** scoped to one sub-account — the spec's security scheme description reads: *"Use the Access Token generated with user type as Sub-Account (OR) Private Integration Token of Sub-Account."* Required scope: `contacts.write`.
- **Required header:** `Version: 2021-07-28` (the OpenAPI parameter marks this required with that single enum value).
- **Only required body field: `locationId`** (the sub-account ID). Everything else — `firstName`, `lastName`, `name`, `email`, `phone`, `address1`, `city`, `state`, `postalCode`, `website`, `timezone`, `dnd`, `tags` (string array), `dateOfBirth`, `country`, `companyName`, `assignedTo`, `source`, and `customFields` — is optional per the `CreateContactDto` schema.
- **Response:** `201` on success; `400`/`401`/`422` documented as error responses (exact error bodies not extracted).

There is also **`PUT /contacts/upsert`**, which the marketplace docs describe as create-or-update based on duplicate matching, governed by the location's "Allow Duplicate Contact" setting.

### 3.2 No public "submit a form" write endpoint

The Forms API v2 ([forms.json](https://raw.githubusercontent.com/GoHighLevel/highlevel-api-docs/main/apps/forms.json)) exposes exactly three operations, **all read-only or auxiliary**:
- `GET /forms/` — list forms (scope `forms.readonly`)
- `GET /forms/submissions` — list submissions (scope `forms.readonly`)
- `POST /forms/upload-custom-files` — upload a file to attach to a custom field (scope `forms.write`)

There is no endpoint to create a form submission via API. This means "submit programmatically as if it came from Form X" is not an option through this API; the only ways to create a lead programmatically are Create/Upsert Contact (§3.1) or the Inbound Webhook workflow trigger (§3.3).

Worth noting for the reverse direction (reading back which form a lead came from, for reporting/QA rather than capture): `GET /forms/submissions` accepts a `formId` query parameter to filter submissions down to one specific form, plus `startAt`/`endAt` date bounds and a free-text `q` filter matched against contact name/email/phone/ID (confirmed from the `FormsParams` parameters in [forms.json](https://raw.githubusercontent.com/GoHighLevel/highlevel-api-docs/main/apps/forms.json)). So if the Inquiry and Plan-request forms are built as two separate GHL-hosted/embedded forms (each with its own `formId`), their submissions are independently queryable via this endpoint without any extra tagging — reinforcing the "which form" half of §5.2's mechanism 1 at the API level, not just inside workflows.

### 3.3 Inbound Webhook workflow trigger (an alternative ingestion path)

GHL workflows can start from a **premium "Inbound Webhook" trigger** ([help: How to use the Inbound Webhook Workflow Premium Trigger](https://help.gohighlevel.com/support/solutions/articles/48001237383-how-to-use-the-inbound-webhook-workflow-premium-trigger-)):

- Creating the trigger generates a unique webhook URL (regenerating it — by deleting and recreating the trigger — issues "a new URL with a different ending ID").
- Accepts **`POST`, `GET`, or `PUT`**, body as a JSON object; keys must be single strings without spaces (camelCase/snake_case).
- The following workflow step ("Create/Update Contact") maps incoming JSON keys to contact fields, **including custom fields**; **an email or phone number is mandatory** in the payload to create/find a contact.
- **No documented authentication mechanism** — the article names no API key, signature, or IP-allowlist option; the only stated security control is regenerating the URL if it leaks. This is worth flagging explicitly for a public marketing-site integration, since anyone who obtains the URL can post arbitrary contacts.
- Arrays are not supported in custom values sent through this path.

### 3.4 Outbound webhooks (not the direction this issue cares about, noted for completeness)

GHL separately supports outbound webhooks (an app/workflow action sending data *out* of GHL) and a fixed catalog of webhook event payloads for Marketplace apps, including `ContactCreate` — whose documented payload shape confirms `customFields` is delivered as `[{id, value}]` on the *outbound* side too ([ContactCreate.md](https://raw.githubusercontent.com/GoHighLevel/highlevel-api-docs/main/docs/webhook%20events/ContactCreate.md)). There is no `FormSubmit`/`FormSubmission` event in GHL's published webhook-event catalog (checked against the full list in the docs repo) — form-submission-driven automation is done via the in-app "Form Submitted" workflow trigger (§5.2), not a webhook event.

---

## 4. Custom fields

### 4.1 Configuration

Contact-level custom fields are configured under **Settings → Custom Fields** ([help: How to Create and Use Custom Fields within HighLevel](https://help.gohighlevel.com/support/solutions/articles/48001161579-how-to-use-custom-fields)). Supported field/data types (confirmed against the `CreateCustomFieldsDTO.dataType` enum in [custom-fields.json](https://raw.githubusercontent.com/GoHighLevel/highlevel-api-docs/main/apps/custom-fields.json)): `TEXT`, `LARGE_TEXT`, `NUMERICAL`, `PHONE`, `MONETORY`, `CHECKBOX`, `SINGLE_OPTIONS`, `MULTIPLE_OPTIONS`, `DATE`, `TEXTBOX_LIST`, `FILE_UPLOAD`, `RADIO`, `EMAIL`.

Key modeling rule from the help doc: **a field is created as either a "Contact" field or an "Opportunity" field, and cannot be switched afterward.** For "which plan is this lead interested in," a Contact-level `SINGLE_OPTIONS` or `RADIO` field (with one option per plan) is the natural fit, since it needs to travel with the contact record regardless of pipeline stage.

### 4.2 Creating a field via API

`POST /custom-fields/` (auth: Bearer token, same scheme as Contacts). Relevant `CreateCustomFieldsDTO` properties, taken directly from the schema:

- `locationId`, `name`, `dataType` (enum above), `description`, `placeholder`, `showInForms` (boolean)
- `options` — array, required for `SINGLE_OPTIONS`/`MULTIPLE_OPTIONS`/`RADIO`/`CHECKBOX`/`TEXTBOX_LIST` types — this is where a fixed list of plan names would live for a `RADIO` "Plan interested in" field.
- `allowCustomOption` (boolean) — for `RADIO` fields, lets a submitted value outside the predefined list be recorded without becoming a permanent new option.
- `fieldKey`/`objectKey` — documented specifically for **Custom Objects** (`custom_object.{objectKey}.{fieldKey}` naming); the schema doesn't show a distinct "contact vs. opportunity" selector property in this DTO, which suggests that distinction is made via a different property not captured in the fetched excerpt, or via the object type context of the call — **flagged as a gap**, see §7.

### 4.3 Setting a field's value on a contact

Confirmed from `CreateContactDto.customFields` in [contacts.json](https://raw.githubusercontent.com/GoHighLevel/highlevel-api-docs/main/apps/contacts.json): it's an array of objects matching one of several per-type schemas (`TextField`, `LargeTextField`, `SingleSelectField`, `RadioField`, `NumericField`, `MonetoryField`, `CheckboxField`, `MultiSelectField`, `FileField`). The common shape, taken directly from the `TextField` schema:

```json
{ "id": "6dvNaf7VhkQ9snc5vnjJ", "key": "my_custom_field", "field_value": "My Text" }
```

`id` is required; `key` (the field's configured key) is accepted as an alternative way to address the field. So a Plan-request submission sent via API would include something like `customFields: [{ "key": "plan_interested_in", "field_value": "Studio Package" }]`, once that field is created per §4.2.

### 4.4 Reading field definitions

`GET /custom-fields/object-key/{objectKey}` and `GET /custom-fields/{id}` exist to look up field IDs/keys/options at build time (e.g., to resolve the "Studio Package" option's exact stored value before sending it).

---

## 5. Source attribution

### 5.1 Native attribution model

GHL's help documentation on attribution ([help: Understanding Attribution Source (Ad Reporting)](https://help.gohighlevel.com/support/solutions/articles/48001219997-understanding-attribution-source-ad-reporting-)) describes a contact-level model with:

- **First Attribution** and **Latest Attribution**, each capturing standard UTM parameters (`source`, `medium`, `campaign`, `content`, `keyword`, `matchtype`) plus platform click IDs (`campaign_id`, `ad_group_id`, `ad_id`, `gclid`, `wbraid`, `gbraid`, `msclkid`).
- Traffic is bucketed into one of nine categories (Paid Search, Paid Social, Organic Search, Social Media, Referrals, Direct Traffic, Others, CRM UI, Third-Party) using rules against the referring domain and UTM values.
- **Critical scoping statement from the article: attribution capture requires "a HighLevel Form, Survey, Calendar, Chat Widget and Order Form"** as the originating action — i.e., this attribution model is tied to GHL-native conversion surfaces, reinforcing that using GHL's own hosted/embedded forms (rather than a bespoke Next.js form posting to the Contact API) is what turns this attribution machinery on for free.

### 5.2 Distinguishing "Inquiry form" vs. "Plan-request form (Plan X)"

Two independent, natively-supported mechanisms, confirmed from official docs:

1. **Which form.** Workflows have a **"Form Submitted"** trigger with a **"Form Is"** filter that targets one specific form by name from a dropdown of the account's forms, plus a **"Form Type"** filter (Normal / Chat Widget / Survey / All) ([help: Workflow Trigger – Form Submitted](https://help.gohighlevel.com/support/solutions/articles/155000002550-workflow-trigger-form-submitted)). This alone distinguishes "Inquiry" from "Plan request" submissions without extra configuration, since they'd be two different forms.
2. **The hidden `source` field.** Every form/survey can carry a hidden **Source** element with a default value set in the builder; critically, that default can be overridden per-link with a query parameter — `https://link.gohighlevel.com/widget/form/[ID]?source=alternative_source` — and the override value flows through to the contact record as **"Contact Source"** ([help: Source Field in Forms and Surveys](https://help.gohighlevel.com/support/solutions/articles/155000001506-source-field-in-forms-and-surveys)). This is the natively documented way to stamp a submission with something more specific than "which form" — e.g., a single Plan-request form could be linked to from three different plan pages with three different `?source=` values.

### 5.3 Getting a specific *custom field* (not just `source`) populated dynamically, e.g. "Plan X"

This is the part relevant to "GHL needs to know which plan/package," if the plan is meant to live in its own custom field rather than being encoded into the generic `source` string. Two documented approaches, both with caveats:

- **URL parameter → matching form field key.** GHL's own doc states the mechanism plainly: *"GoHighLevel checks the URL. If it finds a parameter with a name that matches a form field's custom field name, it uses that value to prefill the field automatically"* ([help: URL Parameters in Websites and Funnels](https://help.gohighlevel.com/support/solutions/articles/155000005722-url-parameters-in-websites-and-funnels)). Concretely: give the "Plan interested in" custom field the key `plan`, then link to the Plan-request form as `.../widget/form/[ID]?plan=Studio+Package`. This is documented behavior, not a workaround.
- **Custom HTML/JS field population inside the embedded form.** A separate official article shows manually setting a field's DOM value and dispatching an `input` event so the value is captured on submit: `document.getElementsByName('customFieldId')[0].value = myData; document.getElementsByName('customFieldId')[0].dispatchEvent(new Event("input"))` ([help: Populate Custom Fields and capture in submission using Custom HTML/Javascript Logic](https://help.gohighlevel.com/support/solutions/articles/155000003043-populate-custom-fields-and-capture-in-submission-using-custom-html-javascript-logic)). This requires knowing the field's internal DOM `id`/`name` (found via browser inspector) and is inherently more brittle than the URL-parameter method, and depends on the embed exposing a same-document DOM (i.e., may not work if the embed is a cross-origin iframe, which the article doesn't address).
- **Caveat on reliability:** GHL's own public feature-request board has an open item, "URL parameters to fill custom fields in forms" (96 votes, opened December 2022, status **"Under review"** as of this research) ([ideas.gohighlevel.com](https://ideas.gohighlevel.com/forms/p/url-parameters-to-fill-custom-fields-in-forms)) — commenters describe the *existing* mechanism as inconsistent for certain field types and on mobile. This is a customer feedback board, not documentation, but it's GHL's own official channel and directly bears on how much to trust §5.3's URL-parameter mechanism at production reliability.
- **Simplest alternative that sidesteps all of the above:** make "plan" a visible (or clearly-labeled) field with pre-set options **on the Plan-request form itself** (radio/dropdown per §4.1), and pass the plan by pre-selecting the browser-side option when the form is opened, or — more robustly — run **one Plan-request form per plan**, distinguished purely via the "Form Is" workflow filter (§5.2, mechanism 1), avoiding dynamic-value injection into the embed entirely. This is a design trade-off for the decision-makers, not a GHL fact, and is only stated here to make clear it's an option the primary sources support without any of the brittleness above.

---

## 6. Next.js-specific constraints

- **No official Next.js/React guide.** Searched `developers.gohighlevel.com`, `marketplace.gohighlevel.com/docs`, and `help.gohighlevel.com` — none returned a framework-specific integration guide.
- **CORS on the public API:** Not stated on any fetched primary-source page. `marketplace.gohighlevel.com/docs` (landing page) makes no statement about CORS or browser-origin restrictions ([marketplace.gohighlevel.com/docs](https://marketplace.gohighlevel.com/docs/)). Regardless of CORS, calling `POST /contacts/` (or any authenticated endpoint) directly from browser JS would require shipping the Bearer token (OAuth token or Private Integration Token) to the client, which the token-security guidance in GHL's own Private Integrations doc argues against implicitly by treating the token as a secret to be "shared securely with developers" and rotated on a schedule ([help: Private Integrations – Everything You Need to Know](https://help.gohighlevel.com/support/solutions/articles/155000003054-private-integrations-everything-you-need-to-know)). **Conclusion (reasoned, not directly documented by GHL): any direct Contact-API integration should be called from a Next.js server context** (Route Handler / Server Action), not client components, so the token never reaches the browser. This is an inference from the credential-handling documentation, not a stated GHL constraint — flagged as such.
- **Rate limits** (from `marketplace.gohighlevel.com/docs`, corroborated by multiple search results returning the same figures): **100 requests / 10 seconds** burst and **200,000 requests/day**, both scoped per Marketplace app per Location (sub-account); responses include `X-RateLimit-*` headers; exceeding the limit returns `429`. For a marketing-site lead form this ceiling is far beyond realistic traffic and is a non-issue.
- **Auth model recap relevant to a Next.js server integration:** a **Private Integration Token** (a static, non-expiring-until-rotated Bearer token scoped to one sub-account, created from the GHL UI, non-recoverable after creation, rotatable with a 7-day overlap window) is the documented fit for "internal purposes... only 1 sub-account at a time" ([help: Private Integrations – Everything You Need to Know](https://help.gohighlevel.com/support/solutions/articles/155000003054-private-integrations-everything-you-need-to-know); [marketplace.gohighlevel.com/docs/Authorization/PrivateIntegrationsToken/](https://marketplace.gohighlevel.com/docs/Authorization/PrivateIntegrationsToken/)) — as opposed to full OAuth 2.0, which GHL's docs describe as required for a tool "intended for the GHL Community or multiple sub-accounts," with tokens that auto-refresh daily. A single studio with one GHL sub-account calling its own CRM from its own Next.js backend is squarely the Private-Integration-Token use case.
- **Required header on every v2 call:** `Version: 2021-07-28` (confirmed as a required OpenAPI parameter on the Create Contact operation; other resources in the same repo may pin different version strings — check the specific endpoint's spec before building).

---

## 7. Open questions / gaps

- **Exact embed snippet markup (iframe vs. script-injected widget) is not published in the help article text** — the embedding-options article documents *behavior* (layouts, triggers, modal sizing) but not the literal HTML/JS GHL generates. This should be pulled from a live GHL account's "Integrate" tab before any Next.js implementation work starts, so CSP and script-loading strategy can be decided against the real markup rather than assumption.
- **No official statement on CSP requirements, iframe auto-resize, or SSR/hydration behavior** for embedded forms — none of the fetched pages mention these terms. Section 6's Next.js guidance is inference, not a documented GHL position.
- **No official statement on the public API's CORS policy.** Whether `services.leadconnectorhq.com` sends `Access-Control-Allow-Origin` headers permitting browser calls was not found in any primary source; several third-party community threads (Zapier community, n8n community — not cited as facts above, since they're not primary) describe backend-proxy patterns consistent with the token-secrecy inference in §6, but that inference should be verified against actual API responses (e.g., an OPTIONS preflight) before deciding on client vs. server call placement.
- **`CreateCustomFieldsDTO` doesn't show an explicit "Contact vs. Opportunity" object-type selector property** in the schema excerpt pulled from `custom-fields.json` — the help-center article states plainly that a field is one or the other and can't be switched, but the exact API property that sets this wasn't identified in the fetched schema. Worth re-checking the full schema (`components/schemas/CreateCustomFieldsDTO` in [custom-fields.json](https://raw.githubusercontent.com/GoHighLevel/highlevel-api-docs/main/apps/custom-fields.json)) or the `GET /locations/get-custom-fields` docs page directly in-product before building.
- **`marketplace.gohighlevel.com/docs` pages render via client-side JS**, and several individual endpoint pages (e.g., the rendered Create Contact and Custom Fields V2 landing pages) returned mostly UI chrome rather than full request/response bodies to the fetch tool used for this research. Where that happened, this doc relied on the raw OpenAPI JSON in GHL's public GitHub repo instead, which is more authoritative (it's the source the docs site is built from) but means a few UI-only details (e.g., in-app copy, exact error-response bodies for 400/401/422) weren't independently verified.
- **No page required a logged-in GHL account to view** — nothing in this research hit a paywall or auth-gated doc; all cited pages were publicly fetchable.
- **The "URL parameter → custom field" prefill mechanism (§5.3) has an open, unresolved feature request on GHL's own feedback board** questioning its reliability for exactly this use case (passing a dynamic value in via URL). Treat it as documented-but-not-guaranteed until tested against the account's actual GHL plan/version.
