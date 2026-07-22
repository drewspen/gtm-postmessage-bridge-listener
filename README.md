# GTM postMessage Bridge for Cross-Origin iframes

A Google Tag Manager container recipe that bridges `dataLayer` events across an
`<iframe>` origin boundary using `window.postMessage()` — built on GTM Sandboxed
JavaScript Custom Templates rather than hand-rolled inline JavaScript, and
parameterized so it's portable across sites.

Blog write-up: *postMessage Bridge for iframes in GTM — Sandboxed, Parameterized,
and Client-ID Aware* — https://drewspen.blogspot.com/

Container export: [`gtm-postmessage-bridge-listener.json`](./gtm-postmessage-bridge-listener.json)

## The Problem

A third-party iframe embedded on a page (a Pardot form, a HubSpot form, a
scheduling widget, a payment step) runs its own GTM container in its own
browsing context with its own `dataLayer`. Nothing that happens inside the
iframe is visible to the parent page's GTM container, and vice versa. If a
visitor clicks "Submit" inside the embedded form, the parent site's GA4
property never hears about it unless something deliberately carries that
event across the origin boundary.

## Credit and Inspiration

This recipe is a variant of Simo Ahava's original GTM `postMessage`
implementation. I'm using his approach successfully in a live Pardot
implementation:

- Video: https://youtu.be/g5J0aUA2Um8
- Article: [How Do I Use the postMessage Method With Cross-Site iframes? — Simo Ahava](https://www.teamsimmer.com/2023/05/02/how-do-i-use-the-postmessage-method-with-cross-site-iframes/)

His article is the clearest explanation of the underlying mechanics — origin
checking, `evt.source` validation, and why a sender/receiver pair of GTM tags
is the right shape for the problem.

## Why This Variant

Inspired by Simo's original and earlier work, this recipe reworks the
implementation with three goals:

1. **Less reliance on Custom HTML / hand-written JavaScript.** Nearly all
   sender and receiver logic lives in GTM Sandboxed JavaScript Custom
   Templates instead of inline `<script>` tags.
2. **More reliance on Sandboxed JavaScript.** Template code executes inside
   GTM's own sandbox and is not subject to CSP `script-src` SHA-256 hashing —
   it's already covered by the `www.googletagmanager.com` allowance a site
   needs regardless.
3. **Fully parameterized configuration.** Target origin, namespace, event
   allow/exclude lists, the iframe's element ID, and the wrap key are all
   template fields, not literal values buried in a script body. Porting the
   recipe to a new site or iframe means editing template fields, not
   rewriting JavaScript.

Reducing reliance on Custom HTML shrinks — but does not eliminate — the CSP
SHA-256 surface. Two small Custom HTML tags remain, one per side, because
sandboxed JavaScript's `access_globals` permission cannot reach predefined
browser globals (`top`, `window`, `parent`, etc.) directly — a Custom
Template cannot call `window.top.postMessage()` itself:

- **Child site:** `postMessage Bridge - Sender Bootstrap` — performs the
  actual `window.top.postMessage()` call.
- **Parent site:** `postMessage Bridge - Bootstrap Listener` — registers the
  single `window.addEventListener('message', …)` handler.

Both scripts' literal text is fixed and configuration-free (everything they
need is read from plain `window` globals written by their companion Custom
Template), so each SHA-256 hash is computed once and stays valid indefinitely
regardless of how you reconfigure origins, namespaces, or allow-lists.

## Passing the Client ID

This recipe also carries the parent page's GA4 `client_id` into the iframe,
via the `_ga` query-string parameter on the iframe's `src`. This reuses the
URL-decoration mechanism from an earlier recipe (see [Related Recipes](#related-recipes-reused-in-this-container)),
aimed at `<iframe src>` instead of an outbound `<a href>`.

Passing the client ID is principally for use with the GTM configuration
settings inside the iframe's own GTM container. Reading that value back out
on the child side gives the child site's GA4 config tag the option to use
the parent's `client_id` instead of generating its own — which gives the
receiving property the opportunity to associate the iframe's traffic with
the same `pseudo_user_id` as the parent site. **It is not the same session**,
and it is not full cross-domain session stitching — but it is enough to
connect embedded-content activity back to the same underlying visitor
identity, where the receiving side chooses to use it.

## Parent Site vs. Child Site — One Download, Two Destinations

Everything in this export is labeled either **Parent Site** or **Child
Site**. Elements marked **Parent Site** belong in the GTM container installed
on the page that *hosts* the `<iframe>`. Elements marked **Child Site** —
along with the shared Google Tag configuration settings that set `client_id`
from the URL query string — belong in the GTM container installed *inside*
the iframe's own page.

Both halves are combined into a single container JSON purely for convenience
of downloading and importing in one pass. To run both halves inside one
shared container (rather than importing into two separate containers), do
both of the following:

1. Add a hostname condition to the **Parent Site** triggers so they fire only
   when `{{Page Hostname}}` equals the parent site's host.
2. Add a hostname condition to the **Child Site** triggers so they fire only
   when `{{Page Hostname}}` equals the child/iframe site's host.

Without one of those two approaches, the Parent-Site and Child-Site logic
will both attempt to evaluate on every page load in whichever container you
import them into. This is harmless — the sender finds no matching dataLayer
events on a page with no iframe events firing, and vice versa — but wasteful
and confusing to debug in GTM Preview mode.

## What's Inside the Container

Five folders, twelve tags, six triggers, fourteen variables, and ten custom
templates.

### Folders

| Folder | Side | Contains |
|---|---|---|
| `postMessage Bridge Sender` | Child | Sender Configuration template tag, Sender Bootstrap Custom HTML tag, diagnostic tag, all-events custom trigger, required (unattached) Click trigger, referral-host constant. |
| `UTM & URL Processing` | Child | Writes UTM query-string values to session cookies on the child page, plus the `_ga` query-string reader/validator that feeds the GA4 config tag's `client_id` override. |
| `Analytics` | Shared | GA4 initialization tag, Browser Client ID variable, Measurement Stream ID constant, Google Tag Shared Configuration Settings variable used by the child-site GA4 config tag. |
| `postMessage Bridge Receiver` | Parent | Receiver template tag, Bootstrap Listener Custom HTML tag, diagnostic tag, Initialization trigger, custom event trigger that catches the bridged `iframe.gtm.click` event. |
| `UTM & URL Processing` | Parent | URL Decoration Config, its "set global" tag, DOM listener/scanner tag, target-host constant, UTM cookie-pair reader, trigger pair that decorates the iframe's `src` with UTM values and the GA4 `client_id`. |

### Tags

| Tag | Side | Fires On | Purpose |
|---|---|---|---|
| `postMessage Bridge - Sender Configuration` | Child | Custom Event, regex `.*` | Scans new `dataLayer` entries since last run, drops `excludeEvents` (default `gtm.js, gtm.dom, gtm.load`), keeps only `eventAllowlist` (`gtm.click` as exported), wraps each event under `namespace`, and stages the result on `window.__gtmPmOutbox` / `window.__gtmPmTargetOrigin`. A per-target cursor in template storage prevents re-sending already-staged events. |
| `postMessage Bridge - Sender Bootstrap` | Child | Same trigger, sequenced after Sender Configuration | Reads the staged outbox/target-origin and calls `window.top.postMessage(JSON.stringify(item), origin)` for each item — the one call sandboxed JS cannot make itself. |
| `Diagnostic - Is postMessage Alive` | Child | Same trigger (paused by default) | **Tag note:** "diagnostic tag. turn it on to see in the dataLayer if the postMessage processing is alive." Logs outbox contents, target origin, and whether the page is running inside an iframe. |
| `Write UTMs to session cookies` | Child | Initialization | UTM-cookie writer scoped to `{{Page Hostname}}`, session-duration by default. |
| `Google Analytics Initialization` | Shared | Initialization | Standard GA4 config tag, routed to `{{Measurement Stream ID}}`, with the Shared Configuration Settings variable applied. |
| `postMessage Bridge - Receiver` | Parent | Initialization | Writes `window.__gtmPmConfig` (allowed origins + iframe element ID) and defines `window.__gtmPmReceiver()`, which validates origin/namespace, de-duplicates an exact repeat of the previous payload, and pushes the unwrapped event to the parent `dataLayer`. **Tag note:** "note the importance of correctly coding the https://{{postMessage Target Host}}, the tag element you are tracking - iframe, and the kinds of events you want to receive from the iframe - the gtm.click events. these must all be correct for the solution to work." |
| `postMessage Bridge - Bootstrap Listener` | Parent | Initialization, sequenced after Receiver | Registers the single `window.addEventListener('message', …)` handler; checks origin against `__gtmPmConfig.allowedOrigins`, optionally confirms `evt.source` matches the tracked iframe's `contentWindow`, and hands off to `window.__gtmPmReceiver()`. |
| `Diagnostic - Is postMessage Alive` | Parent | All Pages (paused by default) | **Tag note:** "diagnostic tag. turn it on to see in the dataLayer if the postMessage processing is alive." Logs every raw incoming message, polls for the receiver config/function to appear, and watches `dataLayer.push` for `iframe.`-prefixed events. |
| `Set URL Decoration Config Global` | Parent | Setup tag only | Writes `{{URL Decoration Config}}` to `window.gtmUrlDecoratorConfig` before the listener tag runs. Reused from the URL-decoration recipe. |
| `URL Decoration Listener` | Parent | DOM Ready | Scans the DOM for elements (including `<iframe src>`) pointing at a permitted host; pushes `urlDecorationTargetFound` if one is found. |
| `Decorate Target URLs` | Parent | Custom Event: `urlDecorationTargetFound` | Appends UTM cookie pairs and the GA4 `client_id` pair onto the iframe's `src`. |
| `Populate form … EXAMPLE` (×2) | Both, paused | DOM Ready | Illustrative only — shows one way a downstream form could read passed-through `_ga`/`utm_campaign` cookie values into hidden fields. Not required for the bridge itself. |

### Triggers

| Trigger | Side | Type | Notes |
|---|---|---|---|
| `postMessage Bridge - Custom Event - All Events (regex .*)` | Child | Custom Event, `{{_event}}` matches `.*` | Deliberately unfiltered at the trigger level — event filtering happens inside the Sender Configuration template instead, so bridged events are changed by editing a tag field, not a trigger regex. |
| `Click - Necessary to Create dataLayer gtm.click` | Child | Click, unattached to any tag | **Trigger note:** "if you are sending gtm.click from the iframe to the parent site, you must have at least the trigger for gtm.click in the iframe container - even if it is not attached to a tag. the absence of the trigger means the absence of the gtm.click event in the dataLayer and resultingly no iframe.gtm.click will be sent by postMessage." GTM only pushes `gtm.click` into the dataLayer when at least one Click trigger exists in the container. |
| `DOM Ready - All Pages` | Parent | DOM Ready | Unconditional as exported — fires on every parent page. **Change this** — see below. |
| `Click from iframe - iframe.gtm.click` | Parent | Custom Event, equals `iframe.gtm.click` | **Trigger note:** "the custom event sent by postMessage from the iframe. note that the components tie back to the 'iframe' and 'gtm.click' configuration parameters in the child / iframe. you now have the full payload of the iframe gtm.click in the parent dataLayer. this can be altered for anything captured in the child / iframe based on the event_name in the child / iframe dataLayer." Attach downstream GA4 event tags here. |
| `URL decoration target found` | Parent | Custom Event, equals `urlDecorationTargetFound` | Fires the decorator tag only once the DOM scan has found a qualifying element. |
| `Window Loaded` | Shared | Window Loaded | Utility trigger from the Analytics folder scaffolding; not wired to a tag in this export. |

### Key Variables

| Variable | Side | Role & Notes |
|---|---|---|
| `Referral Host - iframe Parent Site` | Child | Constant, default `drewspen.blogspot.com`. **Note:** "enter the host name of the parent / referral site here." Feeds Sender Configuration's `targetOrigin` as `https://{{this}}`. |
| `postMessage Target Host` | Parent | Constant, default `postmessage-drewspen.blogspot.com`. **Note:** "the host name of the iframe source. note that not only this must match, but the iframe id must also be configured correctly to match in the tag configuration." Feeds the Receiver's `allowedOrigins` and the URL Decoration Config's `permittedHosts`. |
| `Obtain _ga client id from URL query string` | Child | Reads the `_ga` query-string parameter off the child page's own URL; falls back to `{{Browser Client ID}}` when absent. |
| `URL Query String _ga Fix` | Child | **Note:** "necessary so that if the _ga value is blank or empty it is converted to a true undefined. a blank or empty will be used to 'erase' the client_id. an undefined will allow for GA to generate a value." A Regex Table validates the resolved value against the GA4 client-ID shape (`NNNNNNNNNN.NNNNNNNNNN`) before trusting it. |
| `Google Tag Shared Configuration Settings` | Child | Sets the GA4 config tag's `client_id` to `{{URL Query String _ga Fix}}`, `send_page_view` to `false`, `debug_mode` to the Preview-only wrapper. This is the mechanism that lets the child iframe's GA4 hits carry the parent's client ID. |
| `Root Domain` / `Root Domain - old` | Shared | **Note:** "Root domain of .blogspot.com did not work for Google Blogger. I did not want to do .blogger.com, so changed to page URL host name." Cookies scope to `{{Page Hostname}}`, not a computed eTLD+1 — same decision as the earlier UTM/URL-decoration recipes. |
| `Debug Mode (Preview Only)` | Shared | **Note:** "Wraps the built-in {{Debug Mode}} variable so the debug_mode config parameter is OMITTED (undefined) on normal traffic rather than explicitly sent. Some GA4/gtag implementations key off the parameter's mere presence, not its value, and omitting it avoids sending a needless param on every hit." |

### Custom Templates

| Template | Used By | Purpose |
|---|---|---|
| `postMessage Bridge - Sender Configuration` | Sender Configuration tag (Child) | Stages allow-listed dataLayer events for postMessage delivery. |
| `postMessage Bridge - Receiver` | Receiver tag (Parent) | Validates and unwraps incoming postMessage payloads into the parent dataLayer. |
| `Get Root Domain` ([mbaersch](https://github.com/mbaersch/get-root-domain)) | `Root Domain - old` variable | Utility; not wired into active tags — the active `Root Domain` variable uses `{{Page Hostname}}` directly for Blogger compatibility. |
| `Write site landing & site referrer 2 cookies` | Reused scaffolding | From an earlier recipe; present but not central to this recipe. |
| `Write URL query strings 2 cookies` | `Write UTMs to session cookies` tags | Reads a configurable list of URL query parameters and writes each present one to a cookie. |
| `Set URL Decoration Config Global` | Same-named tag (Parent) | Writes a config variable to a window global for a downstream Custom HTML tag to read. |
| `Get GA Client ID Pair` | URL decoration flow | Packages `{{Browser Client ID}}` as a `{name: '_ga', value: clientId}` pair. |
| `URL Decoration Config` | `URL Decoration Config` variable | Builds the JSON object consumed by the URL Decoration Listener — permitted hosts and enabled element/attribute targets. |
| `URL Link Decorator` | `Decorate Target URLs` tag | Rewrites matching element attributes with decoration pairs appended as query-string parameters. |
| `Get Cookie Pairs by Name List` | `Get UTM Cookie Pairs` variable | Reads a configured list of cookie names and returns them as `{name, value}` pairs. |

## How the Full Round Trip Works

Outbound (parent → child) and inbound (child → parent) run independently,
but together they form a loop:

### 1. Outbound: decorating the iframe's `src` (Parent Site)

Reuses the URL-decoration recipe wholesale, aimed at the iframe element
instead of an outbound link. On DOM Ready, the listener scans the page; if it
finds an `<iframe>` whose `src` host matches `{{postMessage Target Host}}`,
it fires the decorator, which appends the parent's UTM cookie values and its
GA4 `client_id` (as `_ga`) onto the iframe's `src`.

### 2. Child side picks up the client ID

The child page's own GA4 config tag reads the `_ga` query parameter (with
the validation and fallback described above) and uses it as its own
`client_id`, instead of letting `gtag.js` generate a fresh one. The two
properties now share a `client_id` — though not a session.

### 3. Sender: staging events inside the iframe

The all-events custom trigger fires Sender Configuration on every dataLayer
push. It walks new dataLayer entries, drops `gtm.js`/`gtm.dom`/`gtm.load`,
keeps only `gtm.click` (as configured), tags each event with page URL and
title, and stages the wrapped result on `window` globals. Sender Bootstrap
runs immediately after and calls
`window.top.postMessage(JSON.stringify(item), 'https://<parent-host>')`.

### 4. Receiver: unwrapping events on the parent

At Initialization, the Receiver tag writes `window.__gtmPmConfig` and defines
`window.__gtmPmReceiver()`. Bootstrap Listener registers the single `message`
listener for the page. When a message arrives, it confirms the origin is
allowed and (if configured) that `evt.source` is really the tracked iframe's
`contentWindow`, then calls the receiver function, which re-checks origin,
confirms the namespace prefix, drops an exact repeat of the previous
payload, and pushes the unwrapped event (`iframe.gtm.click`, wrapped under
`postMessageData`) into the parent's `dataLayer`.

### 5. Downstream on the parent

The `Click from iframe - iframe.gtm.click` Custom Event trigger listens for
exactly that event name. Attach any GA4 event tag (or additional trigger
conditions on the bridged payload's fields) to turn "someone clicked inside
the embedded form" into a first-class parent-site GA4 event.

## What You Need to Change Before Using This

**Host names.** `Referral Host - iframe Parent Site` (Child, default
`drewspen.blogspot.com`) must be the real host of the page that will embed
your iframe. `postMessage Target Host` (Parent, default
`postmessage-drewspen.blogspot.com`) must be the real host the iframe's
`src` points at. Both origin checks are exact, case-insensitive hostname
comparisons — no wildcards, no path matching.

**The tracked iframe element.** The Receiver tag's `iframeElementId` field
ships as `iframe-form-tracking`. This must exactly match the `id` attribute
on the actual `<iframe>` element in your parent page's HTML — if it doesn't,
the receiver's `evt.source` check silently rejects every message. Leave it
blank only if you're comfortable accepting messages from any frame at the
allowed origin, regardless of which element it came from.

**Which events get bridged.** The Sender Configuration tag's
`eventAllowlist` ships as `gtm.click` only. Add other dataLayer event names
(comma-separated) to bridge additional events, e.g. a custom `form_submit`
event the child container pushes. The Receiver's own `eventAllowlist` is a
second, independent gate on the parent side — both allow-lists have to agree
for an event to land in the parent's dataLayer.

**Trigger scoping — fire only where your iframe actually is.** As exported,
`DOM Ready - All Pages` (Parent) has no conditions and fires on every page,
and the Child-side all-events trigger is unconditional inside whatever
container it's imported into. Neither is a correctness problem by itself —
the decorator only acts when the DOM scan finds something, and the sender
only stages events that match its allow-list — but both are worth
tightening. On the parent, add a Page Path or Page URL condition to
`DOM Ready - All Pages` so the DOM scan only runs on template(s) that
actually embed your tracked iframe. On the child, if running a shared
container per the two-hostname approach above, add the child hostname
condition to the same trigger. Scope both sides to the pages where your
target iframes are actually found.

## Related Recipes Reused in This Container

- [GTM URL Decoration to Transfer Attribution](https://drewspen.blogspot.com/2026/07/gtm-url-decoration-to-transfer.html) —
  the permitted-hosts, config-global, DOM-scan-then-decorate pattern used
  here to decorate the iframe's `src`.
- [UTM & URL Query String 2 Cookies](https://drewspen.blogspot.com/2026/06/utm-url-query-string-2-cookies.html) —
  the session-cookie writer that keeps UTM values available on the parent
  page beyond the original landing page.

## Diagnostics

Both sides ship a paused `Diagnostic - Is postMessage Alive` tag. Enable the
child-side one to confirm the outbox is being staged and the target origin
is set correctly; enable the parent-side one to confirm the listener is
registered, watch every raw incoming message regardless of origin, and see
bridged events land in the dataLayer in real time. Turn both back off before
publishing — they are verbose `console.log` tags meant for GTM Preview mode
only.

## How to Import

1. Download [`gtm-postmessage-bridge-listener.json`](./gtm-postmessage-bridge-listener.json)
   from this repository (or the Google Drive link in the blog post).
2. In GTM, go to **Admin → Import Container**.
3. Upload `gtm-postmessage-bridge-listener.json`.
4. Choose **Merge** (not Overwrite) to preserve your existing container
   setup, and pick a target Workspace.
5. If importing into **two separate containers** (recommended): import the
   whole file into both, then delete the Parent-Site-labeled items from the
   child container and the Child-Site-labeled items from the parent
   container.
6. If importing into **one shared container**: keep everything, but add the
   hostname trigger conditions described above.
7. Update `Referral Host - iframe Parent Site` and `postMessage Target Host`
   with your real hostnames.
8. Update the Receiver tag's `iframeElementId` to match your actual
   `<iframe id="…">`.
9. Update `Measurement Stream ID` with your real GA4 Measurement ID on both
   sides.
10. Scope the `DOM Ready - All Pages` trigger (and, if applicable, the
    child-side custom event trigger) to the pages that actually contain your
    tracked iframe.
11. Confirm consent settings — the Sender Configuration and Sender Bootstrap
    tags require `functionality_storage` as exported; adjust to match your
    CMP setup.
12. Open GTM Preview mode on both the parent page and the child/iframe page.
    Turn on both diagnostic tags temporarily. Click something inside the
    iframe and confirm: `gtm.click` fires in the child dataLayer → the
    outbox stages an item → `postMessage` fires → the parent's listener
    receives it → `iframe.gtm.click` lands in the parent dataLayer.
13. Turn the diagnostic tags back off and publish.

## License / Contributing

Container JSON and this documentation are published for reuse and
adaptation. If you extend the recipe — additional bridged event types, a
different namespace per iframe, or support for more than one tracked iframe
on the same parent page — open a pull request or issue.
