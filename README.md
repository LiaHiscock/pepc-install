# The `<install>` Element

The `<install>` element is a declarative HTML element that allows web developers to offer
installation of web applications directly from a page. It renders a user-agent-controlled button
whose text and iconography are determined by the browser, providing a strong signal of user intent
and protection against spoofing. The element is part of the
[Permission Element](https://wicg.github.io/PEPC/permission-elements.html) family, sharing the
same security model, styling restrictions, and validation infrastructure.

The `<install>` element is one of two entry points to web app installation initiated by a
website. The other is the [Web Install API][api] (`navigator.install()`), which provides a
promise-based JavaScript entry point. Both share a single backend implementation — manifest
fetch, validation, consent UI, and error taxonomy.

## Authors

- [Lia Hiscock](https://github.com/LiaHiscock) ([Microsoft](https://microsoft.com/))
- [Mike West](https://github.com/mikewest) ([Google](https://google.com/))

## Participate

- [Issue tracker](https://github.com/WICG/install-element/issues)
- [Chromestatus](https://chromestatus.com/feature/5152834368700416)

## Explainers

There are currently two explainers for the `<install>` element, reflecting two different attribute
designs for how the element identifies the app to install.

### Install-URL design (`installurl` / `manifestid`) -- Origin Trial

**[explainer-install-url.md](./explainer-install-url.md)**

- **Status:** Active Origin Trial in Chrome and Microsoft Edge through M152.
- **Attributes:** `installurl` (a page URL) and `manifestid` (the app's computed manifest id).
- **How it works:** On activation, the element loads the page at `installurl` in the background,
  discovers its `<link rel="manifest">`, fetches and validates the manifest, then presents the
  installation dialog.
- This is the currently-shipping design. Developers in the Origin Trial should use this
  explainer as their reference.
- This design will reach end of life at its Origin Trial conclusion.

### Manifest-URL design (`manifest` / `id`) -- Incubating

**[explainer-manifest-url.md](./explainer-manifest-url.md)**

- **Status:** Incubating. Not yet shipping in any Origin Trial.
- **Planned Attributes:** `manifest` (a manifest URL) and `id` (the app's computed manifest id).
- **How it works:** On activation, the element fetches the manifest directly from the supplied
  URL, skipping the intermediate page load.
- This design eliminates the document-fetch step, reducing latency and removing the
  cross-origin HTML parse surface. It is expected to replace the install-URL design at the
  close of the Origin Trial.

## Stakeholder Feedback / Opposition

- Chromium: Positive (implementing, in Origin Trial)
- WebKit: Opposed
- Mozilla: No signals

## References & Acknowledgements

This proposal builds on the [Permission Element (PEPC)][pepc] infrastructure and the
[Web Install API][api].

Many thanks for valuable feedback and advice from:

- Daniel Murphy
- Rob Paveza
- Lu Huang
- Alex Russell
- Arthur Sonzogni

[api]: https://github.com/MicrosoftEdge/MSEdgeExplainers/blob/main/WebInstall/explainer.md
[pepc]: https://github.com/WICG/PEPC/
