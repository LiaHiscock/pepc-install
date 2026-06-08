# `<install>` element — Manifest-URL design

> **Status: Incubating.** This document describes a forward-looking design that is **not**
> currently shipping in any Origin Trial. For the `<install>` element's currently-shipping
> attribute shape (`installurl` / `manifestid`), available in OT through Chrome/Edge M152,
> see [README.md](./README.md).

## Authors

- [Lia Hiscock](https://github.com/LiaHiscock) ([Microsoft](https://microsoft.com/))

<!-- TODO: add co-authors as design coalesces. -->

## Participate

- [Issue tracker](https://github.com/WICG/install-element/issues)
- [Chromestatus (Origin Trial)](https://chromestatus.com/feature/5152834368700416)

## Status of this Document

This document is a starting point for engaging the community and standards bodies in
developing collaborative solutions fit for standardization. As the design coalesces, this
document is expected to replace [README.md](./README.md) at the close of the element's
Origin Trial (Chrome/Edge M152), at which point the OT design will move to an archived
directory.

- This document status: **Incubating**
- Expected venue: [W3C Web Incubator Community Group](https://github.com/WICG)
- Companion to: [README.md](./README.md) (the currently-shipping OT shape)

## Table of contents

<!-- TODO: generate. -->

## Why this document exists

The `<install>` element's Origin Trial exposes an `installurl` / `manifestid` attribute
design. The [Web Install API][api] — which shares the element's backend implementation —
has since pivoted to a manifest-URL design, taking the app's manifest URL directly instead
of inferring it from a page URL.

For the element and the API to genuinely share a backend, the element's attributes need to
align with that pivot. This document incubates the aligned design.

## Relationship to other proposals

- [Web Install API][api] (MSEdgeExplainers) — defines the install algorithm, manifest
  fetch and validation pipeline, consent UI contract, error taxonomy, and cross-origin /
  sandbox / activation gates. **Normative for backend behavior.** This document references
  but does not re-specify those algorithms.
- [README.md](./README.md) (this repo) — defines the element's currently-shipping OT
  shape (`installurl` / `manifestid`). **Normative for the Origin Trial only.** Will be
  archived at OT close.
- **This document** — defines the element's forward-looking shape and what is
  element-specific on top of the WebInstall backend.

## User-Facing Problem

<!--
TODO: Same problem space as README, but rewrite the cross-origin catalog example to use
manifest URLs. The same-origin (no-attribute) and suite-of-apps cases are unaffected.

Suggested beats:
- Same-origin install button: `<install></install>` (unchanged).
- Cross-origin app catalog: catalog supplies the manifest URL of each listed app.
- Suite of apps: same as today, with manifest URLs instead of page URLs.
-->

### Goals

<!-- TODO: largely the same as README. Add: "Match the manifest-URL shape of the shared backend
so the element and API stay in lockstep." -->

### Non-goals

<!-- TODO: same as README, plus: "Support both `installurl`/`manifestid` and the new attributes
in the long run. The OT shape will be removed when this design ships." -->

## Proposed Approach

> **Note** — this proposal assumes familiarity with the
> [permission element spec][pepc-spec], which outlines in detail the element's behavior,
> including styling and activation restrictions, error handling, etc.

### Element attributes

- `manifest` — URL of the web app manifest to install. The same URL a caller would pass
  to [`navigator.install({ manifest })`][api]. If omitted, the page's own manifest is used
  (same-origin install).
- `id` — Optional. The manifest id of the app to install. Must be an absolute URL when
  supplied. If omitted, the manifest at `manifest` must declare an `id` field.

```html
<!-- Install the current page. -->
<install></install>

<!-- Install a specific app by its manifest URL. -->
<install manifest="https://app.example.com/manifest.webmanifest"></install>

<!-- Install a specific app, asserting its manifest id for safety. -->
<install manifest="https://app.example.com/manifest.webmanifest"
         id="https://app.example.com/?source=catalog">
  <a href="https://app.example.com/" target="_blank">Open App</a>
</install>
```

### Activation behavior

On activation, the element invokes the install algorithm defined in [Web Install API][api]
with the supplied `manifest` URL and optional `id`. The backend's algorithms for manifest
fetch, validation, consent UI, and error mapping apply unchanged.

Element-specific outcomes are surfaced via the PEPC event model (`onpromptaction`,
`onpromptdismiss`, `onvalidationstatuschange`), and validation failures are reflected via
`isValid` / `invalidReason`. The mapping between backend errors and element events is
identical to the OT shape; see [README.md](./README.md#element-behavior) for the table.

### IDL

```webidl
[Exposed=Window]
interface HTMLInstallElement : HTMLElement {
  [HTMLConstructor] constructor();

  [CEReactions, ReflectURL] attribute USVString manifest;
  [CEReactions, ReflectURL] attribute USVString id;
};
HTMLInstallElement includes InPagePermissionMixin;
```

## What's element-specific vs. what defers to WebInstall

| Topic | Defined here | Defined in WebInstall |
|---|---|---|
| Element shape, attributes, IDL | Yes | — |
| PEPC mixin (`isValid`, `invalidReason`, validation events) | Yes | — |
| Fallback content | Yes | — |
| Launch-when-installed behavior | Yes (open — pending side-channel resolution, see [#17](https://github.com/WICG/install-element/issues/17)) | — |
| Manifest fetch / parse / validate | References | Yes (normative) |
| Consent UI contract | References, may add element-flow mockups | Yes (normative) |
| Cross-origin, sandbox, user-activation gates | References, adds PEPC visibility overlay | Yes (normative) |
| Error taxonomy | Maps backend errors onto element events | Yes (normative) |
| `manifestId` privacy contract (not returned) | References | Yes (normative) |

## Migration from the OT shape

For developers using the Origin Trial (Chrome/Edge M143–M152) shape:

| OT attribute | Replacement | Notes |
|---|---|---|
| `installurl="https://app.example/"` | `manifest="https://app.example/manifest.webmanifest"` | Supply the manifest URL directly. The element no longer fetches the page to discover the manifest. |
| `manifestid="https://..."` | `id="https://..."` | Same semantics; renamed for parity with `navigator.install()`. |
| `<install></install>` (no attributes) | `<install></install>` | Unchanged. The page's own manifest is used. |

The document-fetch step performed by the OT shape (load `installurl`, find
`<link rel="manifest">`, then invoke the install backend) is removed under this design.
Latency is reduced and the cross-origin parse surface is eliminated.

## Alternatives Considered

### Keep the OT shape (`installurl` / `manifestid`)

<!-- TODO: write up. Pros: developer ergonomics for catalogs (origin vs. manifest URL).
Cons: doesn't share a backend with `navigator.install()` end-to-end; adds a cross-origin
HTML parse surface; stale-manifest-URL risk for the API doesn't disappear, it just moves
to a different code path. -->

### Support both attribute sets

<!-- TODO: write up. Pros: smoother migration. Cons: two-ways-to-do-it antipattern; doubles
validation surface; tends to entrench. -->

### Anchor-based (`<a rel="install" href="manifest URL">`)

<!-- TODO: same content as README's anchor-based alternative section. -->

## Open Questions

<!--
TODO: this section is the main reason this doc is separate. Things to think about:

- Should `manifest` be required for cross-origin installs and optional for same-origin?
  Or always optional (defaulting to the page's own manifest)?
- How does the element surface manifest-fetch errors before activation? The OT shape
  validates eagerly via `invalidReason`; with manifest-URL we could do the same, but it
  costs a fetch on every page load with an `<install>` element.
- Does the element pre-fetch the manifest to render app metadata (name, icon) in the
  button, or stay user-agent-styled like today?
- Same iframe / sandbox question as README — likely the same answer.
- Does `<install>` participate in any new permission policy now that backend gates change?
-->

## Accessibility, Internationalization, Privacy, and Security Considerations

<!--
TODO: most of this carries from README. Specifically address:

- Privacy: the manifest-URL design removes the cross-origin HTML parse surface that the
  OT shape had. Note this as a positive shift.
- Security: the element's PEPC visibility / occlusion / cooldown checks still apply. The
  backend's cross-origin and sandbox gates also still apply (defined in WebInstall).
- Accessibility / I18n: unchanged from README — both are properties of the rendered
  button, which is identical between the two designs.
-->

## Stakeholder Feedback / Opposition

<!-- TODO: this is a fresh design; fill in as positions land. -->

## References & Acknowledgements

This document builds on the [Permission Element (PEPC)][pepc] infrastructure and the
[Web Install API][api]. Substantial credit to the OT shape's authors and contributors
(see [README.md](./README.md#references--acknowledgements)) whose work this design
iterates on rather than replaces.

[api]: https://github.com/MicrosoftEdge/MSEdgeExplainers/blob/main/WebInstall/explainer.md
[pepc]: https://github.com/WICG/PEPC/
[pepc-spec]: https://wicg.github.io/PEPC/permission-elements.html
[mixin]: https://wicg.github.io/PEPC/permission-elements.html#permission-mixin
