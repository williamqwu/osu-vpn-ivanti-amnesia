# Is this an Ivanti-wide problem or specific to OSU?

**Conclusion: the mechanism is a general design of the Ivanti client, not something
specific to OSU. OSU's deployment only determines "how often it flares up" and "what
a flare-up looks like." And macOS users are worst off — the only official workaround
is Windows-only.**

Below I break it down layer by layer, marking the strength of evidence for each
point.

---

## Layer 1: the Ivanti client — general, independent of the school

This layer is determined entirely by the app's own architecture; no gateway
configuration can change it:

| Fact | Evidence | Strength |
|---|---|---|
| The embedded browser has only one store, shared by all login attempts | measured paths on this machine (see root-cause §2) | direct measurement |
| The UI has no "clear browsing data" control at all | the app's actual UI + no corresponding menu item in the binary | direct measurement |
| Closing the browser window != clearing state | Ivanti docs: "*Ivanti Secure Access Client will close the embedded browser, once the SAML authentication is done.*" | vendor docs |
| Not fully quitting the app means cached cookies get reused | Ivanti docs: "*If Ivanti Secure Access Client is not closed and opened again, the client authenticates the user without prompting for credentials after the first successful authentication using cached cookies.*" | vendor docs |
| Clearing IdP state is hard-wired to SLO succeeding | binary attribute name `webSAMLSloUrlToClearCookies`, state machine `CSLOIdpLogoutHelper` | direct measurement |
| SLO has explicit failure branches | binary string `Failed to start WebLogin Session for logout` | direct measurement |
| SLO depends on the embedded browser (so you can't have both) | Ivanti docs: "*Ivanti Secure Access Client supports Single Logout only when embedded browser is enabled.*" | vendor docs |
| Routine cookie clearing is URL-scoped + optional session-only, and doesn't cover persistent cookies | `clearCookieforUrl:clearSessionOnly:`, etc. | direct measurement |

Those two lines of vendor docs are the crux: **Ivanti itself wrote "reuse cached
cookies unless you fully close the app" as expected behavior.** This isn't a bug,
it's by design — it's just that the design periodically backfires under the
combination of SAML + short-lived IdP conversations.

**There's prior history, too**: Ivanti 22.x has already fixed a cookie-handling bug
in this same embedded browser — cookies were handled incorrectly when multiple realms
(at least one of them SAML) were configured under the same sign-in URL, fixed in
22.3R2. It shows this code has been fragile historically.

> Conclusion: **any "Ivanti/Pulse client + SAML SSO" deployment has this structure.**
> It has nothing to do with OSU.

---

## Layer 2: platform — macOS is worst off

Ivanti provides a connection-set option to change which browser SAML uses:

> "**Enable embedded browser for authentication**: When enabled, Ivanti Secure Access
> Client uses the system default browser for SAML authentication, rather than external
> browser. **This feature is supported only on Windows.**"

(This vendor copy contradicts itself — the option is named "embedded," yet the
description says it uses the "system default browser." The only reliable part is the
last sentence: **this is an admin-side option, and it's Windows-only.**)

This means:

- **Windows**: admins can push SAML login out to the system browser. Users can then
  rescue themselves with their browser's private window / clear-data, and the state
  is no longer owned solely by Ivanti.
- **macOS / Linux**: **no supported workaround.** You must use the embedded browser,
  which means you must live with that shared, unclearable store.

Interestingly, the macOS binary **does contain** external-browser code paths
(`launchDefaultBrowser:`, `Launch External Browser using URL %S`), but there's no
corresponding supported configuration option. So that path is effectively unavailable
to macOS users.

And even if you could turn off the embedded browser, there's a cost: per the docs,
**turning off the embedded browser means no SLO**. It's not a free win.

---

## Layer 3: the OSU deployment — only affects "frequency and appearance"

These are OSU-side, and would differ at another school:

| OSU's configuration | Effect |
|---|---|
| The IdP is Shibboleth | Determines **what the error page looks like**. With Azure AD / Okta the error copy would be entirely different, but the underlying mechanism is the same |
| `__Host-shib_idp_username` has a **1-year** lifetime | IdP-side policy. Lets the IdP skip the username step, so you hit the failure point faster and it's harder to notice the state has gone dirty |
| Duo Universal Prompt (OIDC) | Inserts an extra layer of one-time `state` + `code`, **one more link that can go stale**. This machine's disk cache holds 13 such one-time callback URLs |
| The gateway has SLO enabled | which is why the client attempts SLO at all (this machine's 9 SLO window keys are traces of those attempts). A deployment without SLO configured wouldn't leave these traces, but would leave more residue |
| max session length of 5 hours | Determines how long the gateway-side session takes to expire. **Unrelated to client-side residue** — this is exactly why "waiting 5 hours doesn't help" |
| The connection is user-created (`connection-source: "user"`) | measured from `connstore.dat`. **No admin-pushed connection set is in effect**, so there's no admin-side browser policy or cleanup policy either |

**None of these is the root cause.** They modulate the probability and the
presentation, not the mechanism itself.

---

## Who runs into this

| Risk | Profile |
|---|---|
| **High** | macOS / Linux + Ivanti client + SAML SSO + a habit of leaving it resident without quitting (← this exact machine) |
| **Medium** | Windows, but the admin hasn't enabled the external browser |
| **Low** | Windows + external browser; or anyone who fully quits the app after each use |
| **None** | Ivanti deployments that don't use SAML (local store / LDAP / certificate auth) |

---

## Corroborating evidence and its limits

- Many universities use Ivanti for their campus VPN (e.g.
  [UCSB](https://it.ucsb.edu/ivanti-secure-access-campus-vpn/vpn-service-frequently-asked-questions-faq),
  [UC Davis Library](https://library.ucdavis.edu/vpn/faq/)), and their
  troubleshooting pages commonly include "clear cookies"-style advice.
- Shibboleth's "stale request" is itself a common cross-institution error, with
  dedicated explainer pages at several schools
  ([Illinois](https://shibboleth.illinois.edu/idp/profile/Authn/SAML2/POST/SSO),
  [Caltech](https://www.imss.caltech.edu/services/computers-printers-software/stale-login-error),
  [UMN](https://it.umn.edu/services-technologies/how-tos/known-errors-stale-request)).

**But to be clear about the limits**: those pages all discuss the stale request in a
**regular browser** scenario, and give generic advice like "clear your browser
cookies / don't use the back button / switch networks."

**I found no public document that identifies "the Ivanti embedded browser's
persistent shared store" as one class of cause.** So this document's stance is: the
symptom is well-recognized and widespread, but this specific causal chain was derived
from my own forensics here and does not appear in vendor or university public
materials. Treat that as "not externally verified."

Those pages also mention a cause this document doesn't cover: **complex NAT making the
client appear to come from multiple IPs as seen by the IdP** (Shibboleth can be
configured with `consistentAddress` to bind a session to an IP). That's worth noting
for a VPN client — but it can't explain the remnants found on this machine (7 months
of cache, the torn cookie file, the accumulated SLO window keys).

---

## Points to raise if filing a ticket

For OSU IT (in order of feasibility):

1. **Shorten the lifetime of `__Host-shib_idp_username`.** One year is too long;
   cutting it to a single session or a few days would significantly reduce the window
   in which dirty state gets silently reused.
2. **Check that the Ivanti SP's SLO endpoint is correctly paired in the IdP
   metadata.** This machine's 9 accumulated SLO window keys show the client **keeps
   attempting** SLO; if the endpoint or binding doesn't match, those attempts are just
   spinning.
3. **For the Windows user population, evaluate enabling the external browser.** Mind
   the trade-off: you lose SLO support.
4. **macOS users currently have no server-side fix**, and can only rely on
   client-side habits (fully quitting with ⌘Q when done) or this tool. This is worth
   spelling out in the university's VPN docs — it's the cheapest, most effective step.

For Ivanti (if worth raising): the embedded browser should offer a user-facing "clear
login state" control, or reset the browsing context before initiating each new SAML
authentication. Right now it does neither.

---

## References

- [Ivanti — User Experience (ISAC 22.X)](https://help.ivanti.com/ps/help/en_US/ISAC/22.X/ag-22.X/user_experience.htm)
- [Ivanti — Connection Set Options](https://help.ivanti.com/ps/help/en_US/ISAC/vNow/cg_ics_client/connection-set-options.htm)
- [Ivanti — Embedded Browser Support](https://help.ivanti.com/ps/help/en_US/ISAC/vNow/linux-qsg/embedded-browser-support.htm)
- [Ivanti — Resolved Issues (ISAC 22.X)](https://help.ivanti.com/ps/help/en_US/ISAC/22.X/rn-22.X/Resolved-issues.htm)
- [Ivanti — SAML Single Sign-on (ICS 22.x)](https://help.ivanti.com/ps/help/en_US/ICS/22.x/ag/saml_single_sign_on.htm)
- [UCSB — Ivanti Secure Access campus VPN FAQ](https://it.ucsb.edu/ivanti-secure-access-campus-vpn/vpn-service-frequently-asked-questions-faq)
- [UC Davis Library — VPN FAQ](https://library.ucdavis.edu/vpn/faq/)
- [Caltech IMSS — Shibboleth Stale Request Login Error](https://www.imss.caltech.edu/services/computers-printers-software/stale-login-error)
- [UMN — Known Errors: Stale Request](https://it.umn.edu/services-technologies/how-tos/known-errors-stale-request)
