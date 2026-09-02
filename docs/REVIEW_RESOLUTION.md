# Review resolution addendum

- PR: #1
- Base: `main`
- Resolution scope: the six existing review threads listed below.
- This document is a normative design addendum. It records the accepted resolution contract; it does not claim that implementation or tests have already run.
- Bot review is not retriggered.

## PRRT_kwDOTNkK4M6OZ6o5 — no-network CSP

Problem: `connect-src 'self'` still permits same-origin requests and does not prove that address data cannot be sent.

Resolution: the static no-network runtime MUST use `connect-src 'none'`, `form-action 'none'`, `base-uri 'none'`, and navigation restrictions appropriate to the app. The privacy claim MUST NOT rely on `self` allowing or denying a same-origin endpoint; no same-origin data-send path may be present.

Focused verification before resolving this thread: inspect the delivered policy and runtime code for fetch/XHR/WebSocket/form/navigation paths, and verify no address-data request can be initiated.

## PRRT_kwDOTNkK4M6OZ6o6 — file:// is not a PWA origin

Problem: `file://` is not a secure origin and does not provide normal PWA/service-worker guarantees.

Resolution: documentation and acceptance MUST use a localhost/static-server or other secure HTTP origin for PWA operation. A `file://` launch MAY be documented only as a separate non-PWA fallback with its limitations; it MUST NOT be presented as the PWA acceptance environment.

Focused verification before resolving this thread: launch through the documented local static server, install/register the PWA, and verify offline reload and service-worker scope on that origin.

## PRRT_kwDOTNkK4M6OZ6o8 — dedicated production origin for address data

Problem: GitHub Pages projects can share an origin and therefore share origin-scoped storage; a project path is not privacy isolation.

Resolution: address data MUST remain on a dedicated production origin/domain selected for this product. Do not claim the Pages project path is sufficient isolation. Deployment and data-storage acceptance MUST remain blocked until that dedicated origin is chosen and documented.

Focused verification before resolving this thread: inspect the deployment origin and storage scope, then verify that unrelated projects cannot share the address-data origin.

## PRRT_kwDOTNkK4M6OZ6o- — year-scoped print exclusions

Problem: manual print exclusions are promised but no year-scoped data field, UI, or acceptance path is defined.

Resolution: add a year-scoped field such as `printExclusionsByYear` to the contact/settings model, expose it through the UI, and make the print filter read the selected year’s value. The design MUST define behavior for absent, empty, and populated year entries.

Focused verification before resolving this thread: configure exclusions for two years and assert that printing each year applies only that year’s setting; test the UI persistence path.

## PRRT_kwDOTNkK4M6OZ6pA — encoding confidence and override

Problem: valid UTF-8 decoding does not prove that the input’s original encoding was UTF-8.

Resolution: import MUST retain original bytes until confirmation, produce a candidate encoding/confidence result, and show a preview. The user MUST be able to select a manual override when confidence is low or the preview is wrong; successful strict UTF-8 decoding alone MUST NOT be treated as certainty.

Focused verification before resolving this thread: import representative UTF-8, legacy-encoding, and ambiguous samples and assert preview, confidence/fallback, and override behavior.

## PRRT_kwDOTNkK4M6OZ6pC — visible PWA update handling

Problem: reopening the app does not guarantee that an updated service worker is active or that the user sees the latest app.

Resolution: the service-worker update path MUST expose a visible reload/update prompt, coordinate activation, and provide a deterministic acceptance path for the user to reload into the new version. “Reopen later” alone is not the update UX contract.

Focused verification before resolving this thread: install version N, publish N+1, observe the update event/prompt, accept reload, and assert the active cache/version changes.

## Verification status

The checks above are required acceptance criteria for implementation. This addendum intentionally reports no test result and no implementation-complete status.