# User-Initiated Installation of a Web Application - Manifest-URL Design

## Authors

- [Lia Hiscock](https://github.com/LiaHiscock) ([Microsoft](https://microsoft.com/))

## Participate

- [Issue tracker](https://github.com/WICG/install-element/issues)
- [Chromestatus (Origin Trial)](https://chromestatus.com/feature/5152834368700416)

## Status of this Document

This document is a starting point for engaging the community and standards bodies
in developing collaborative solutions fit for standardization.

- This document status: **Incubating**
- Expected venue: [W3C Web Incubator Community Group](https://github.com/WICG)
- Current version: this document

## Table of contents

<!-- TODO: generate. -->

## Introduction

The `<install>` element is a declarative HTML element that allows web developers
to offer installation of web applications directly from a page. It renders a
user-agent-controlled button whose text and iconography are determined by the
browser, providing a strong signal of user intent and protection against spoofing.
The element is part of the [Permission Element](https://wicg.github.io/PEPC/permission-elements.html)
family, sharing the same security model, styling restrictions, and validation
infrastructure.

## Relationship to other proposals

- [Web Install API][api] (MSEdgeExplainers) -- defines the backing install
  algorithm, manifest fetch and validation pipeline, consent UI contract, error
  taxonomy, and cross-origin / sandbox / activation gates.
  **Normative for backend behavior.** This document references but does not
  re-specify those algorithms.
- [Permission Element spec][pepc-spec] (WICG) -- defines the base infrastructure
  that `<install>` is built on: `HTMLCapabilityElementBase`, `InPagePermissionMixin`,
  the blocker system, intersection observer visibility checks, and styling
  restrictions. This infrastructure ships in production via the
  [`<geolocation>`][geolocation] (Chrome 144) and `<usermedia>`elements.
- **This document** -- defines install-specific properties/behaviors on top of
  the CapabilityElementBase and WebInstall backend.

| If you're asking… | See |
|---|---|
| How do I add an install button to my page? | This doc |
| Why won't the element activate? | [Permission Element spec][pepc-spec] |
| What styling am I allowed to apply? | [Permission Element spec][pepc-spec] |
| Why did the manifest fail to fetch or parse? | [Web Install API][api] |
| What events fire after activation? | This doc |

## User-Facing Problem

Think about all the websites you use regularly - email, online shopping, social
media, streaming sites, etc. For most users, this requires launching a browser
and clicking or typing to get to those sites every time. Web applications enable
developers to provide native, "app-like" experiences to end users while building
on the trust set by their browser. However, for end users there's no standard,
cross-platform way to acquire web applications. The process of distributing and
installing web apps is both fragmented and limited:

- Each browser has different, often hidden, entry points for installation
  (address bar icons, menu items, prompts).
- Users may not know that a web app exists for the site they're visiting, or
  that "installation" is even possible on the web.
- Developers have no standard declarative mechanism to present an install action
  to users.
- Cross-origin installation (e.g. an app catalog installing apps from other
  sites) has no built-in web platform support.

### Goals

- Provide a **declarative** way to install web applications, requiring no
  JavaScript for basic usage.
- Give the **user agent control** over the button's content and presentation,
  providing a trustworthy signal of user intent.
- Support both **same-origin** and **cross-origin** installation scenarios.
- Offer **progressive enhancement** through fallback content for browsers that
  don't support the element.
- Integrate with the existing [Permission Element][pepc-spec] infrastructure for
  consistent security, styling, and validation behavior.

### Non-goals

- Replace [navigator.install()][api]. The imperative and declarative APIs serve 
  complementary use cases and share a backend implementation.
- Define what "installation" means. This varies by platform and browser.
- Install arbitrary web content that is not an app (the target must have a
  manifest file).

## Use Cases

### Install me! (Same origin install button)

A web app can present an install button on its own page. This is the simplest
case -- no attributes are needed:

```html
<install></install>
```

### Cross-origin app catalog

A web-based app store or catalog can install apps from other origins by supplying
the manifest URL of each listed app:

```html
<install manifest="https://music.youtube.com/manifest.webmanifest"
         id="https://music.youtube.com/?source=pwa">
</install>
```

### Suite of web apps

A productivity suite can install related apps from the same origin:

```html
<install manifest="https://suite.example/mail/manifest.webmanifest">Install Email</install>
<install manifest="https://suite.example/calendar/manifest.webmanifest">Install Calendar</install>
<install manifest="https://suite.example/tasks/manifest.webmanifest">Install Tasks</install>
```

## Proposed Approach

> As noted above in related proposals, this approach assumes familiarity with
> the [permission element spec][pepc-spec], which outlines in detail the
> element's behavior, including styling and activation restrictions,
> error handling, etc.

A declarative `<install>` element that renders a button whose content
and presentation is controlled by the user agent. Similar to other
[permission elements][pepc] (e.g. [`<geolocation>`][geolocation]),
the user agent's control over the element's content means that it can
make plausible assumptions about a user's contextual intent. Users who
click on a button labeled "Install" are unlikely to be surprised if an
installation flow begins.

### Element content

The element renders standardized text and iconography controlled by
the user agent:

<img alt='A button whose text reads "Install", with an icon signifying the action of installation.' src='./install-icon.png'>

### Element attributes

- `manifest` -- Optional. URL of the web app manifest to install. If
  omitted, the current page's manifest is used (same-origin install).
- `id` -- Optional. The manifest id of the app to install. Must be an
  absolute URL when supplied. If omitted, the manifest at `manifest`
  must declare an `id` field.

```html
<!-- Install the current page. -->
<install></install>

<!-- Install a specific app by its manifest URL. -->
<install manifest="https://app.example.com/manifest.webmanifest"></install>

<!-- Install a specific app whose manifest does not declare an `id` -->
<install manifest="https://app.example.com/manifest.webmanifest"
         id="https://app.example.com/?source=catalog"></install>
```

### Element fallback content

If the user agent doesn't support installation, fallback content can be rendered:

<img alt='A hyperlink reading "Launch YouTube Music".' src='./install-not-supported.png'>

```html
<install manifest="https://music.youtube.com/manifest.webmanifest"
         id="https://music.youtube.com/?source=pwa">
  <a href="https://music.youtube.com/" target="_blank">
    Launch YouTube Music
  </a>
</install>
```

### Activation behavior

On activation, the element invokes the install algorithm defined in
[Web Install API][api] with the optionally supplied `manifest` and `id`. The
backend's algorithms for manifest fetch, validation, consent UI, and error
mapping apply unchanged.

Where `navigator.install()` uses promise rejections with `DOMException` names,
the `<install>` element surfaces outcomes through two mechanisms: pre-click
validation via the [InPagePermissionMixin][mixin], and post-click results via
`InstallResultEvent` (see [Error handling](#error-handling--debuggability) below).

Example consent UI for users to review security-sensitive fields such as the app
name, origin, and icon before installation proceeds:

![An installation prompt for YouTube Music.](./dialog-ytmusic.png)

## Error handling / debuggability

The element surfaces errors through two distinct mechanisms, reflecting the
difference between pre-activation validation and post-activation installation results.

### Pre-activation: `InPagePermissionMixin`

The element inherits the standard [InPagePermissionMixin][mixin] validation
interface (`isValid`, `invalidReason`, `onvalidationstatuschange`) from the
[Permission Element spec][pepc-spec]. These cover [presentation restrictions][pepc-security]
common to all capability elements (visibility, styling, occlusion, temporal
cooldowns) and are not install-specific.

A violation of these restrictions prevent the element from being activated
(either temporarily or permanently) and surface as an error in the
Developer Tools > Issues tab.

### Post-activation: `InstallResultEvent`

Errors and outcomes that occur *after* the user clicks (manifest fetch failures,
parse errors, manifest `id` mismatches, user cancellation, success) are surfaced
via a dedicated `InstallResultEvent`. The result is a property on the event
object -- not on the element -- because the install flow is asynchronous and a
result tied to the element could be overwritten by a subsequent attempt before
the handler runs.

```js
element.addEventListener('installresult', (event) => {
  console.log(event.result);
});
```

The exact set of result values exposed to the installing origin is under
discussion — see the [Web Install API explainer's privacy section][api-privacy]
for the full analysis.

[api-privacy]: https://github.com/MicrosoftEdge/MSEdgeExplainers/blob/main/WebInstall/explainer.md#what-result-information-should-be-exposed-to-the-caller

## Alternatives Considered

### Install-URL design (`installurl` / `manifestid`)

An alternative design where the element takes a page URL (`installurl`) instead
of a manifest URL, and loads the page in the background to discover its
`<link rel="manifest">`.

This has the advantage of simpler developer ergonomics for catalogs (page URLs
may be more stable than manifest URLs), but it introduces a variety of security,
privacy, and performance concerns. See [explainer-install-url.md](./explainer-install-url.md)
for the full design.

### Declarative `<a>`-based

`<a href="manifest_url" rel="install">`

The Web Install API proposal considered a different declarative approach.
This gives the user agent less control over the content and presentation but has
the advantage of built-in progressive enhancement. The `<install>` element approach
was chosen because it gives the user agent full control over the button's rendering,
consistent with the PEPC model.

See the [API][api] proposal for detailed analysis.

### Alternative element names

Names like `<pwa>` or `<webapp>` could more broadly describe the range of behavior
(install and launch). `<install>` was preferred because launching is a
privacy-preserving fallback rather than the primary purpose.

## Open Questions

### Can the element pre-fetch the manifest to render app metadata?

Rendering the app name, origin, or icon in the element would provide a stronger
signal of user intent but introduces performance, UX, security, and accessibility
complications (e.g. long app names, icon contrast ratios, button layout changes).

### Will this work with WebXR/WebGL scenarios?

No, this is a known limitation of the element-based approach.

> The [HTML in Canvas](https://github.com/WICG/html-in-canvas) proposal could
> make this possible, but additional consideration is needed to avoid
> privacy/security leaks. See [#9](https://github.com/WICG/install-element/issues/9).

### How does it behave in iframes?

Currently restricted to top-level browsing contexts for security purposes.
Same-origin iframes are unlikely to pose a risk and may be supported in the future.

### How does it behave in sandboxed contexts?

Currently disabled in all sandboxed contexts. If a use case for installing from
a sandbox presents itself, a strict allow-list can be implemented. This decision
is open to reevaluation.

### What text should be in the button?

The button's rendering is implementation-defined. User agents may combine an
action verb with the application's origin, render the app's name if trusted, or
use other approaches. It is worth discussing what considerations user agents
should pay attention to, but specifying the content too precisely would be unhelpful.

### Handling long names and origins

User agents will need to consider how to handle very long words, including
appropriate resizing, eliding, and truncation logic (similar to what the
installation dialog already implements). User agents should apply the same
considerations they use [elsewhere][url-display] for displaying origins and names.

### Does `<install>` participate in any new permission policy?

Subject to the same `WebAppInstallation` permission policy as `navigator.install`.

### What result information should be exposed to the installing origin?

The `InstallResultEvent` result values and `navigator.install()` promise rejections
share the same underlying question: how much should the installing origin learn
about the outcome of an install attempt — particularly in the cross-origin case?

This is a shared backend concern. See the [Web Install API explainer's privacy section][api-privacy]
for the full analysis, including the three options under consideration,
side-channel considerations, and `AbortError` convention alignment.

[api-privacy]: https://github.com/MicrosoftEdge/MSEdgeExplainers/blob/main/WebInstall/explainer.md#what-result-information-should-be-exposed-to-the-caller

### What if the app is already installed?

In the future, the user agent could render the element as a "Launch" button when
the app is already installed, following established [launch handler](https://developer.mozilla.org/en-US/docs/Web/API/Launch_Handler_API)
algorithms. User agents must avoid exposing installed status to side-channel
attacks. See [#17](https://github.com/WICG/install-element/issues/17) for
details.

## Accessibility, Localization, Privacy, and Security Considerations

### Accessibility

The element renders as a button and inherits standard button accessibility
semantics, including keyboard navigation and focus management. The default
`tabindex` is 0. Screen readers should announce the element's text content
(controlled by the user agent).

### Localization

The element observes the `lang` attribute to select localized text for the button
label (e.g. "Install" in the page's language). App names from manifests may also
be localized based on browser language.

### Privacy

- The manifest-URL design fetches the manifest file directly, eliminating the
  cross-origin HTML parse surface present in the install-URL design. This is a
  positive privacy shift -- fewer cross-origin resources are loaded before user
  consent.
- The element does not reveal whether an app is installed. The "Launch" state
  has been removed pending mitigation of a width side-channel discovered during
  security review. See [#17](https://github.com/WICG/install-element/issues/17).
- Web apps are not installable from private/incognito modes. User agents must
  ensure they don't expose information indicating that's why installation failed
  (e.g. failing immediately, which could hint that no manifest was fetched).
- Cross-origin installation does not grant any permissions to the installing
  origin. Each installed app has its own independent set of permissions.

### Security

- The element inherits
  [PEPC presentation restrictions][pepc-security]: visibility checks, contrast
  ratio requirements, font size bounds, occlusion detection, and temporal cooldowns.
  These prevent clickjacking and ensure the user can see and understand what
  they're clicking.
- Cross-origin installation requires the user agent to present the target origin
  clearly. Sites can restrict their installation to same-origin by checking
  `Sec-Fetch-Dest: manifest` and `Sec-Fetch-Site` headers.
- The element is disabled in cross-origin subframes, fenced frames, and all
  sandboxed contexts.

## Stakeholder Feedback / Opposition

- W3C TAG Review: PENDING
- Browser Standards Positions:
  - Chromium: [Supportive/Implementing](https://chromestatus.com/feature/5183481574850560)
  - Mozilla: [mozilla/standards-positions#1179](https://github.com/mozilla/standards-positions/issues/1179)
  - WebKit: [WebKit/standards-positions#463](https://github.com/WebKit/standards-positions/issues/463)

## References & Acknowledgements

This proposal builds on the [Permission Element (PEPC)][pepc] infrastructure and
the [Web Install API][api].

Many thanks for valuable feedback and advice from:

- Daniel Appelquist
- Amanda Baker
- Marcos Cáceres
- Diego Gonzalez
- Lu Huang
- Alex Russell
- Arthur Sonzogni
- Daniel Murphy
- Howard Wolosky

[api]: https://github.com/MicrosoftEdge/MSEdgeExplainers/blob/main/WebInstall/explainer.md
[pepc]: https://github.com/WICG/PEPC/
[pepc-spec]: https://wicg.github.io/PEPC/permission-elements.html
[pepc-security]: https://github.com/WICG/PEPC/blob/main/explainer.md#security-abuse
[geolocation]: https://github.com/WICG/PEPC/blob/main/geolocation_explainer.md
[mixin]: https://wicg.github.io/PEPC/permission-elements.html#permission-mixin
[activation-behavior]: https://dom.spec.whatwg.org/#eventtarget-activation-behavior
[activate-geo]: https://wicg.github.io/PEPC/permission-elements.html#ref-for-dom-inpagepermissionmixin-features-slot%E2%91%A1%E2%93%AA
[url-display]: https://chromium.googlesource.com/chromium/src/+/HEAD/docs/security/url_display_guidelines/url_display_guidelines.md
