# User-Initiated Installation of a Web Application

## Authors

- [Lia Hiscock](https://github.com/LiaHiscock) ([Microsoft](https://microsoft.com/))
- [Mike West](https://github.com/mikewest) ([Google](https://google.com/))

## Participate

- [Issue tracker](https://github.com/WICG/install-element/issues)
- [Chromestatus](https://chromestatus.com/feature/5152834368700416)

## Status of this Document

This document is a starting point for engaging the community and standards bodies in developing
collaborative solutions fit for standardization.

- This document status: **Active**
- Expected venue: [W3C Web Incubator Community Group](https://github.com/WICG)
- Current version: this document
- Origin Trial: The `<install>` element is available as an [Origin Trial](https://developer.chrome.com/docs/web-platform/origin-trials/) in Chrome and Microsoft Edge.

## Table of Contents

<!-- TODO: generate with doctoc -->

## Introduction

The `<install>` element is a declarative HTML element that allows web developers to offer
installation of web applications directly from a page. It renders a user-agent-controlled button
whose text and iconography are determined by the browser, providing a strong signal of user intent
and protection against spoofing. The element is part of the
[Permission Element](https://wicg.github.io/PEPC/permission-elements.html) family, sharing the
same security model, styling restrictions, and validation infrastructure.

## User-Facing Problem

End users don't have a standard, cross-platform way to acquire web applications. The process of
distributing and installing web apps is both fragmented and limited:

- Each browser has different, often hidden, entry points for installation (address bar icons,
  menu items, prompts).
- Users may not know that a web app exists for the site they're visiting, or that "installation"
  is even possible on the web.
- Developers have no standard declarative mechanism to present an install action to users.
- Cross-origin installation (e.g. an app catalog installing apps from other sites) has no
  built-in web platform support.

### Goals

- Provide a **declarative** way to install web applications, requiring no JavaScript for
  basic usage.
- Give the **user agent control** over the button's content and presentation, providing a
  trustworthy signal of user intent.
- Support both **same-origin** and **cross-origin** installation scenarios.
- Offer **progressive enhancement** through fallback content for browsers that don't support
  the element.
- Integrate with the existing [Permission Element][pepc-spec] infrastructure for consistent
  security, styling, and validation behavior.

### Non-goals

- Replace [navigator.install()][api]. The imperative and declarative APIs serve complementary
  use cases and share a backend implementation.
- Define what "installation" means. This varies by platform and browser.
- Install arbitrary web content that is not an app (the target must have a manifest file).

## Use Cases

### Same-origin install button

A web app can present an install button on its own page. This is the simplest case — no
attributes are needed:

```html
<install>
  <a href="/about">Learn more about our app</a>
</install>
```

### Cross-origin app catalog

A web-based app store or catalog can install apps from other origins:

```html
<install installurl="https://music.youtube.com/"
         manifestid="https://music.youtube.com/?source=pwa">
  <a href="https://music.youtube.com/" target="_blank">
    Launch YouTube Music
  </a>
</install>
```

### Suite of web apps

A productivity suite can install related apps from the same origin:

```html
<install installurl="https://suite.example/docs">Install Docs</install>
<install installurl="https://suite.example/sheets">Install Sheets</install>
<install installurl="https://suite.example/slides">Install Slides</install>
```

## Proposed Approach

> **Note** — this proposal assumes familiarity with the
> [permission element spec][pepc-spec], which outlines in detail the element's
> behavior, including styling and activation restrictions, error handling, etc.

A declarative `<install>` element that renders a button whose content and presentation is
controlled by the user agent. Similar to other [permission elements][pepc] (e.g.
[`<geolocation>`][geolocation]), the user agent's control over the element's content means that
it can make plausible assumptions about a user's contextual intent. Users who click on a button
labeled "Install" are unlikely to be surprised if an installation flow begins.

### Element content

The element renders standardized text and iconography controlled by the user agent:

<img alt='A button whose text reads "Install", with an icon signifying the action of installation.' src='./install-icon.png'>

### Element attributes

`installurl` specifies the document to install. If unspecified, the current site will be
installed.

`manifestid` specifies the computed id of the document to install. If unspecified, the manifest
referenced by the document at `installurl` must have a custom `id` defined. If specified, it must
match the computed `id` of the site to be installed.

```html
<install installurl="https://music.youtube.com/"
         manifestid="https://music.youtube.com/?source=pwa">
  [Fallback content goes here.]
</install>
```

#### Other valid element usages

```html
<!-- Install the current page. -->
<install></install>

<!-- The manifest file at installurl must declare an id. -->
<install installurl="https://reddit.com/">
</install>
```

### Element fallback content

If the user agent doesn't support installation, fallback content is rendered:

<img alt='A hyperlink reading "Launch YouTube Music".' src='./install-not-supported.png'>

```html
<install installurl="https://music.youtube.com/"
         manifestid="https://music.youtube.com/?source=pwa">
  <a href="https://music.youtube.com/" target="_blank">
    Launch YouTube Music
  </a>
</install>
```

### Element behavior

On click, the user agent initiates the installation flow. The backend steps are shared with
[navigator.install()][api] via the same implementation
(see the [background document installation steps][bg-steps] and
[current document installation steps][cd-steps] for the full flow).

The key difference is how results are surfaced. Where `navigator.install()` uses promise
rejections with `DOMException` names, the `<install>` element uses the
[InPagePermissionMixin][mixin] event model:

| Outcome | `navigator.install()` | `<install>` element |
|---|---|---|
| Invalid URL / no manifest / id mismatch | `DataError` rejection | `invalidReason` updated, `onvalidationstatuschange` fired |
| Invalid installurl attribute | `DataError` rejection | Element disabled, `invalidReason` set |
| User cancels / dismisses prompt | `AbortError` rejection | `onpromptdismiss` fired |
| Installation succeeds | Promise resolves with `{ id }` | `onpromptaction` fired |
| No user activation | `NotAllowedError` rejection | Click ignored (element not valid) |
| Called outside main frame | `InvalidStateError` rejection | Element not supported in iframes/sandboxes |

The element can also present a confirmation dialog showing the app name and origin
before installation proceeds:

![An installation prompt for YouTube Music.](./dialog-ytmusic.png)

### Dependencies on non-stable features

- [Permission Elements (PEPC)][pepc-spec] — `<install>` extends the shared
  `HTMLCapabilityElementBase` alongside `<permission>` and `<geolocation>`, inheriting
  the blocker system, intersection observer visibility checks, styling restrictions,
  and `InPagePermissionMixin` interface.

### What if the app is already installed?

> **Note** — The "Launch" state is not currently implemented. During security review,
> a width side-channel was discovered between the "Install" and "Launch" renderings.
> The `IsInstalled` backend code has been removed entirely pending a privacy-safe
> design. See [#17](https://github.com/WICG/install-element/issues/17) for details.

In the future, the user agent could render the element as a "Launch" button when the app
is already installed, following established [launch handler](https://developer.mozilla.org/en-US/docs/Web/API/Launch_Handler_API)
algorithms. User agents must avoid exposing installed status to side-channel attacks.

## Error handling / debuggability

The element surfaces errors through the [InPagePermissionMixin][mixin] interface:

- `isValid` returns whether the element can currently be activated. Returns `false` when the
  element is blocked for any reason (visibility, styling, data errors, etc).
- `invalidReason` returns a string specifying why the element is currently invalid. This includes
  the standard [presentation restrictions][pepc-security] for permission elements, as well as
  install-specific data validation errors (e.g. invalid `installurl`, failed manifest fetch,
  `manifestid` mismatch).
- `onvalidationstatuschange` fires when the validation status changes.
- `onpromptaction` fires when installation completes successfully.
- `onpromptdismiss` fires when the user cancels or dismisses the installation prompt.

Developers don't need to use any of these for simple cases — `<install></install>` and
`<install installurl="..."></install>` work without any event handling.

### Upcoming: `InstallResultEvent`

An `oninstallresult` event with a dedicated `InstallResultEvent` interface is under development
to provide richer result information to developers. This is not yet available.

## IDL

```webidl
[Exposed=Window]
interface HTMLInstallElement : HTMLElement {
  [HTMLConstructor] constructor();

  [CEReactions, ReflectURL] attribute USVString installurl;
  [CEReactions] attribute USVString manifestid;
};
HTMLInstallElement includes InPagePermissionMixin;
```

The [`InPagePermissionMixin`][mixin] is defined as part of the
[Permission Element spec][pepc-spec], and includes the following attributes and events:

- `isValid` — `true` if the element is a valid click target (visible, comprehensible, and
  stable long enough to be understood by the user), `false` otherwise.
- `invalidReason` — an enum specifying why the element is currently invalid, including
  install-specific reasons like data errors from manifest fetching/parsing.
- `initialPermissionStatus` and `permissionStatus` — inherited from the mixin. For `<install>`,
  the permission is always bypassed (the element goes directly to the installation dialog),
  so these reflect `"granted"` and are not meaningful for this element.
- `onpromptaction` — fired when the user completes an installation prompt.
- `onpromptdismiss` — fired when the user cancels or dismisses an installation prompt.
- `onvalidationstatuschange` — fired when the validation status changes.

The element's [activation behavior][activation-behavior] is similar to other permission elements
(e.g. [`<geolocation>`'s activation behavior][activate-geo]): the event must be trusted, the
element must be valid, and then the installation flow is triggered. Unlike `<permission>` and
`<geolocation>`, the `<install>` element always bypasses the permission prompt and goes directly
to the installation confirmation dialog.

The element hooks into the same backend as `navigator.install()`. When clicked, it loads the
`installurl` in the background to obtain the web application manifest and related resources
needed for the installation dialog. The fetching and processing steps follow those defined for
[the "manifest" link type][manifest-fetch]. If a valid manifest is obtained, the installation
dialog is presented. If not, the element reports the error via `invalidReason` and
`onvalidationstatuschange`.

## Open Questions

### Will this work with WebXR/WebGL scenarios?

No, this is a known limitation of the element-based approach.

> The [HTML in Canvas](https://github.com/WICG/html-in-canvas) proposal could make this
> possible, but additional consideration is needed to avoid privacy/security leaks.
> See [#9](https://github.com/WICG/install-element/issues/9).

### Are iframes supported?

Currently restricted to top-level browsing contexts for security purposes. Same-origin iframes
are unlikely to pose a risk and may be supported in the future.

### How does it behave in sandboxed contexts?

Currently disabled in all sandboxed contexts as an interim measure. If a use case for installing
from a sandbox presents itself, a strict allow-list can be implemented. This decision is open to
reevaluation.

### What text should be in the button?

The button's rendering is implementation-defined. User agents may combine an action verb with
the application's origin, render the app's name if trusted, or use other approaches. It is
worth discussing what considerations user agents should pay attention to, but specifying the
content too precisely would be unhelpful.

### Handling long names and origins

User agents will need to consider how to handle very long words, including appropriate resizing,
eliding, and truncation logic (similar to what the installation dialog already implements).
User agents should apply the same considerations they use
[elsewhere][url-display] for displaying origins and names.

## Accessibility, Internationalization, Privacy, and Security Considerations

### Accessibility

The element renders as a button and inherits standard button accessibility semantics, including
keyboard navigation and focus management. The default `tabindex` is 0. Screen readers should
announce the element's text content (controlled by the user agent).

### Internationalization

The element observes the `lang` attribute to select localized text for the button label (e.g.
"Install" in the page's language). App names from manifests may also be localized based on
browser language.

### Privacy

- The element does not reveal whether an app is installed. The "Launch" state has been
  removed due to a width side-channel discovered during security review. See
  [#17](https://github.com/WICG/install-element/issues/17).
- Web apps are not installable from private/incognito modes. User agents must ensure they
  don't expose information indicating that's why installation failed (e.g. by not failing
  immediately, which could hint that no manifest was fetched).
- Cross-origin installation does not grant any permissions to the installing origin.
  Each installed app has its own independent set of permissions.

### Security

- The element inherits [PEPC presentation restrictions][pepc-security]: visibility checks,
  contrast ratio requirements, font size bounds, occlusion detection, and temporal cooldowns.
  These prevent clickjacking and ensure the user can see and understand what they're clicking.
- Cross-origin installation requires the user agent to present the target origin clearly.
  Sites can restrict installation to same-origin by checking `Sec-Fetch-Dest: manifest` and
  `Sec-Fetch-Site` headers.
- The element is disabled in fenced frames and sandboxed contexts.
- The element always bypasses the `web-app-installation` permission prompt — the PEPC
  visibility and activation checks provide sufficient signal of user intent.

## Alternatives Considered

### Web Install API (`navigator.install()`)

The [Web Install API][api] provides an imperative, promise-based approach to installation.
It is currently available as an [Origin Trial](https://developer.chrome.com/docs/web-platform/origin-trials/)
in Chrome and Edge. The `<install>` element and `navigator.install()` are complementary —
they share the same backend implementation and can coexist. The declarative element provides
stronger user-intent signals through the PEPC security model, while the imperative API offers
more programmatic control.

### `<a href="..." rel="install">`

A declarative approach using the anchor element with a `rel="install"` attribute. This gives
the user agent less control over the content and presentation but has the advantage of
built-in progressive enhancement. The `<install>` element approach was chosen because it
gives the user agent full control over the button's rendering, consistent with the PEPC model.

### Alternative element names

Names like `<pwa>` or `<webapp>` could more broadly describe the range of behavior (install
and launch). `<install>` was preferred because launching is a privacy-preserving fallback
rather than the primary purpose.

## Future Design Considerations

### Install by manifest URL?

Should `installurl` be supplemented with a `manifesturl` attribute pointing directly to the
manifest file? This could avoid the overhead of loading the document, but introduces concerns
around manifest spoofing, service worker registration, and stale URLs. See
[#5](https://github.com/WICG/install-element/issues/5) for discussion.

### Should manifest id be required?

Currently, if the developer does not provide a `manifestid` attribute, the manifest at
`installurl` must have an `id` field. See
[#6](https://github.com/WICG/install-element/issues/6).

### Custom information in the button

Rendering the app name, origin, or icon in the element would provide a stronger signal of
user intent but introduces performance, UX, security, and accessibility complications. See
the [WICG discussion](https://github.com/WICG/install-element/issues) for details.

## Stakeholder Feedback / Opposition

- Chromium: Positive (implementing, in Origin Trial)
- WebKit: No signals
- Mozilla: No signals

## References & Acknowledgements

This proposal builds on the [Permission Element (PEPC)][pepc] infrastructure and the
[Web Install API][api].

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
[manifest-fetch]: https://html.spec.whatwg.org/multipage/links.html#link-type-manifest:linked-resource-fetch-setup-steps
[bg-steps]: https://github.com/MicrosoftEdge/MSEdgeExplainers/blob/main/WebInstall/explainer-background-doc.md#background-document-1-param
[cd-steps]: https://github.com/MicrosoftEdge/MSEdgeExplainers/blob/main/WebInstall/explainer-current-doc.md#steps-to-install-the-app
[url-display]: https://chromium.googlesource.com/chromium/src/+/HEAD/docs/security/url_display_guidelines/url_display_guidelines.md
