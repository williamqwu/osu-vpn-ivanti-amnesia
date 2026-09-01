# Root cause

Investigated 2026-09-01, on **Ivanti Secure Access 22.8.2** on macOS
(bundle id `net.pulsesecure.Pulse-Secure`), connecting to the OSU College of
Engineering VPN.

Every "measured" figure in this document comes from read-only forensics on that
machine.

---

## 1. The login chain

```
Ivanti client
   │  ① POST /dana-na/auth/url_default/welcome.cgi
   ▼
vpn.coeit.osu.edu                      Ivanti Connect Secure (SAML SP)
   │  ② 302 → SAMLRequest
   ▼
webauth.service.ohio-state.edu         Shibboleth IdP
   │  ③ 302 → OIDC authorize
   ▼
api-XXXXXXXX.duosecurity.com           Duo Universal Prompt
   │  ④ 302 → /idp/profile/Authn/Duo/2FA/duo-callback?state=<nonce>&code=<authz>
   ▼
Shibboleth IdP → SAMLResponse → gateway → tunnel established
```

The Duo hostname is per-tenant; the real subdomain is masked here as
`api-XXXXXXXX`.

All three hops run inside Ivanti's **embedded WKWebView**, not Safari.

How I can tell: the main process holds `WebCore.framework/.../Localizable.strings`;
the binary contains method signatures for `WKWebView` / `WKWebViewConfiguration` /
`WKNavigationAction` / `WKWindowFeatures`; and the preferences store native
window-position keys like `NSWindow Frame /https:/webauth.service.ohio-state.edu/...`.

> The app bundle also ships a full copy of CEF (`Contents/Frameworks/cefBrowser
> Helper*.app`, `Contents/Plugins/JamUI/cefBrowser.app`, bundle id
> `org.cef.cefsimple`). Ivanti's docs say CEF is there for FIDO U2F. On this machine
> the chain goes through WKWebView: all SSO cookies land in the main app's store, and
> CEF has no separate cookie store.

There is also a LaunchAgent `net.pulsesecure.WebLoginService` (vends a Mach service;
its `WatchPaths` points at `~/Library/Application Support/Pulse Secure/WebLoginService`)
that carries the login session. **Its last exit code was -9 (SIGKILL).**

---

## 2. Storage architecture — the structural source of the problem

The embedded browser has only **one store, shared by every login attempt**, and
Ivanti's UI has **no way to clear it**:

| Store | Path | Lifetime |
|---|---|---|
| Cookie jar | `~/Library/HTTPStorages/net.pulsesecure.Pulse-Secure.binarycookies` | persists across restarts |
| HTTP cache | `~/Library/Caches/net.pulsesecure.Pulse-Secure/Cache.db` + `fsCachedData/` | persists across restarts |
| WebKit data | `~/Library/WebKit/net.pulsesecure.Pulse-Secure/WebsiteData/` | persists across restarts |
| alt-svc | `~/Library/HTTPStorages/net.pulsesecure.Pulse-Secure/httpstorages.sqlite` | persists across restarts |
| Window position | `~/Library/Preferences/net.pulsesecure.Pulse-Secure.plist` | persists across restarts |
| **session cookie** | **app process memory** | **gone only when the process fully exits** |

The last row is the easiest to overlook: Ivanti is a menu-bar-resident app. When you
click Disconnect the window closes and the tunnel drops, but **the process stays
alive** — `JSESSIONID`, `shib_idp_session`, and Shibboleth's conversation state are
all still in memory, and get carried verbatim into the next login.

Ivanti's own docs describe exactly this gap:

> "Ivanti Secure Access Client will close the embedded browser, once the SAML
> authentication is done."

**Closing the window != clearing the state.**

Corroborating this in passing, the docs say:

> "If user resizes the Embedded browser window, size will remain same even if user
> reconnects to Ivanti Secure Access Client."

— which is exactly where those `NSWindow Frame /https:/...` keys in this machine's
preferences come from.

---

## 3. Why state gets left behind

### 3.1 The only mechanism that clears IdP state is SAML Single Logout, and it often fails

The client binary contains a full SLO state machine:

```
CSLOIdpLogoutHelper { m_Hostname, m_SAMLIdPLogoutRequestUrl, m_SLOState }
webSAMLSloUrlToClearCookies / setSAMLSloUrlToClearCookies:
getSAMLIdPLogoutRequestUrl / checkIfSamlLogoutIsComplete:
kPromptTypeSAMLLogout           "prompt for SAMLLogout(IdP logout)"
                                "prompt for SAMLLogout(SA logout)"
```

The flow is `logout.cgi` → send `SLO?SAMLRequest=…` to the IdP. Note that the
attribute is literally named **`webSAMLSloUrlToClearCookies`** — clearing cookies is
hard-wired to SLO succeeding.

The same binary also has the branches for when it fails:

```
"Failed to start WebLogin Session for logout"
"%s() Failed. Not launching the embedded browser for logout"
```

And SLO in turn depends on the embedded browser (official docs: *"Ivanti Secure
Access Client supports Single Logout only when embedded browser is enabled."*) —
which forms a closed loop: **you need the embedded browser to get SLO, and the
embedded browser is exactly the carrier of that never-reset, dirty store.**

### 3.2 The client's cookie clearing is "scope-limited," not a full wipe

```
clearCookieforUrl:clearSessionOnly:
"WebLogin: Clearing all the cookies(clearSessionOnlyCookies:%d)"
"WebLogin: Clearing all the Auth cookies"
"WebLogin: Clearing the cookies for url [%s]"
```

All of it is scoped by URL and gated by a `clearSessionOnly` switch. In other words:
**persistent cookies are outside the normal clearing scope to begin with.** Measured
lifetimes of this machine's persistent cookies:

| Cookie | Domain | Lifetime |
|---|---|---|
| `__Host-shib_idp_username` | `webauth.service.ohio-state.edu` | **1 year** |
| `DSSIGNIN` | `vpn.coeit.osu.edu` | **13 months** |
| `DSBrowserID` | `vpn.coeit.osu.edu` | **13 months** |

The IdP's and gateway's identity markers effectively never expire.

---

## 4. Why Shibboleth throws this particular error

The Shibboleth IdP stores an in-progress authentication flow as a **server-side
conversation**, bound by cookies like `JSESSIONID` / `shib_idp_req_ss` plus the
`execution` token in the URL. The Duo leg is OIDC: `state` is a one-time nonce and
`code` is a one-time authorization code.

When the IdP receives a request whose conversation key / `state` is **no longer in
its table of live sessions**, it returns that "Unable to complete request" page.

The error copy attributes the cause to "you were too slow" and iCloud Private Relay.
The former is just the most common benign cause; the latter is measured off on this
machine (`PrivacyProxyServiceStatus = 0`). The real cause is that the client came
back carrying the previous round's leftovers — and sections 2 and 3 show that those
leftovers are a **structural inevitability**.

### Why waiting out the 5-hour timeout doesn't help

What the timeout expires is the session on the **gateway server**. The leftovers are
on the **client**: the on-disk cookie jar and HTTP cache, plus the session cookie in
the resident process's memory. Five hours has no effect on any of those.

This explains the "I waited and it still didn't work" observation.

---

## 5. Forensic evidence from this machine

| Evidence | Data |
|---|---|
| One-time SSO callbacks in the disk cache | **13** `…/duo-callback?state=<nonce>&code=<authz>`, oldest 2026-02 |
| Cache span | 66 responses, `2026-02-11` → `2026-09-01`, never evicted |
| Trace of hitting the same error page before | the cache holds `webauth…/images/dialog_error@2x.png`, timestamp `2026-03-06 13:32` |
| Accumulated SLO window keys | **9** `NSWindow Frame …/SAML2/Redirect/SLO?SAMLRequest=…` |
| Torn cookie write | orphan file `…binarycookies_tmp_1281.dat` (2026-03-04, 181 days ago) |
| Hard-kill trace | `net.pulsesecure.WebLoginService` last exit code **-9** |
| Long-lived persistent cookies | see the table in §3.2 |

`_tmp_<pid>.dat` is CFNetwork's intermediate file for atomically writing the cookie
jar: write a temp file, then rename. If the process is SIGKILL'd before the rename,
the temp file is left behind. It corroborates `WebLoginService`'s -9: **this app was
hard-killed at least once, right inside the window where the cookie was being flushed
to disk.**

> The tool therefore deliberately **does not send SIGKILL** — it uses only AppleScript
> quit → SIGTERM, and exits with an error if it can't stop the app. It won't
> reproduce the very bug it diagnosed.

### What remains unproven

Exactly which cookie / parameter the IdP rejected at the moment of failure was **not
captured directly** — that would require forensics while the error page is still on
screen. The causal chain above is inferred from the structure plus the remnants, not
observed from a packet capture.

`ivanti-amnesia capture` exists to fill in that missing link: it snapshots the cookie
jar, preferences, and `Cache.db`, and grabs the Pulse / Ivanti / WebLoginService
entries from the unified log.

---

## 6. Appendix: the binarycookies format (reverse-engineered)

For the tool to perform the surgery of "delete only the SSO cookies, keep Duo device
trust," it has to be able to rewrite this file safely. The format is below, **verified
byte-for-byte round-trip identical** on this machine's two jars:

```
file
  'cook'                       4B
  page_count                   4B  big-endian
  page_size[page_count]        4B each, big-endian
  pages                        ...
  checksum                     4B  big-endian
  0x07 0x17 0x20 0x05          4B  magic
  meta_len                     4B  big-endian
  meta                         binary plist (NSHTTPCookieAcceptPolicy)

page
  0x00 0x00 0x01 0x00          4B
  cookie_count                 4B  little-endian
  cookie_offset[count]         4B each, little-endian (absolute offset from page start)
  0x00 0x00 0x00 0x00          4B  separator
  cookie records               ...

cookie record (all little-endian internally)
  0x00  record_size    4B
  0x08  flags          4B   bit0 = Secure, bit2 = HttpOnly
  0x10  domain_offset  4B
  0x14  name_offset    4B
  0x18  path_offset    4B
  0x1C  value_offset   4B
  0x28  expires        8B   double, Mac epoch (2001-01-01)
  0x30  created        8B   double, Mac epoch
  then 4 NUL-terminated strings

checksum = one byte taken every 4 bytes across each page, all summed together
```

Two easy traps (both hit in practice, and both inconsistent with descriptions
floating around online):

1. **Those 4 bytes `00 00 00 00` sit after the offsets array and before the
   records**, not as a "footer at the end of the page" as many sources claim.
2. **The trailing 8 bytes are not a single magic value**, but a 4-byte magic
   `07 17 20 05` plus a 4-byte meta-plist length.

The tool's approach is to **preserve the original record bytes and only add or remove
records**, never re-encoding an individual cookie; before rewriting it runs a
byte-level round-trip self-check, and if that fails it refuses the surgery and asks
you to switch to `--level full`.

---

## References

- [Ivanti — User Experience (ISAC 22.X)](https://help.ivanti.com/ps/help/en_US/ISAC/22.X/ag-22.X/user_experience.htm)
- [Ivanti — Embedded Browser Support](https://help.ivanti.com/ps/help/en_US/ISAC/vNow/linux-qsg/embedded-browser-support.htm)
- [Ivanti — Connection Set Options](https://help.ivanti.com/ps/help/en_US/ISAC/vNow/cg_ics_client/connection-set-options.htm)
- [Ivanti — SAML Single Sign-on (ICS 22.x)](https://help.ivanti.com/ps/help/en_US/ICS/22.x/ag/saml_single_sign_on.htm)
