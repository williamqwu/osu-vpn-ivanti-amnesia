# ivanti-amnesia

> [!WARNING]
> **This repository is archived and is no longer maintained.**
> OSU College of Engineering no longer uses Ivanti VPN, so the deployment this tool
> was built against and tested on no longer exists. Nothing here is being updated,
> and none of it has been re-verified against a current setup.
>
> It is kept for reference: the root cause was never specific to OSU (see
> [Scope](#scope) below), and the reverse-engineered `binarycookies` format in
> [`docs/root-cause.md`](docs/root-cause.md) stands on its own. Read it as a
> snapshot, not as working instructions.

> Make Ivanti Secure Access forget the stale SSO state that jams its SAML login.

A one-shot recovery tool for when Ivanti Secure Access (macOS) SAML login gets
stuck.

The name comes from the root cause: this bug is fundamentally about **the client
remembering things it shouldn't** — leftovers from the previous login get carried,
verbatim, into the next one. The fix is to force it to forget.

## Motivation

My school's OSU CoE VPN uses Ivanti Secure Access. Every so often it fails partway
through login, and there is **no self-healing path**: reconnecting doesn't help,
restarting the app doesn't help, waiting out the timeout doesn't help. Ivanti's UI
has neither a "clear browsing data" control nor any hint that the state has gone
dirty.

Each time, the only way back was to blindly retry until it happened to work. This
tool freezes that blind retry into a single repeatable, reversible command.

## Scope

**Built for, and tested against, exactly one deployment**: OSU College of
Engineering's VPN (`vpn.coeit.osu.edu` -> OSU's Shibboleth IdP -> Duo Universal
Prompt), on macOS, against Ivanti Secure Access 22.8.2. Every measured number in the
docs comes from that one machine.

The root cause, though, is a property of the Ivanti client itself -- one embedded
browser store shared by every login attempt, with no way to clear it from the UI --
and not of anything OSU configured. So the analysis, and most likely the tool itself,
should carry over to **any "Ivanti Secure Access + SAML SSO" deployment**. The
institution's configuration changes how often the failure flares up and what the
error page looks like, not the mechanism.

**That was never tested anywhere but OSU.** Elsewhere, treat it as plausible but
unverified: start with `status` and `fix --dry-run`, both of which are read-only, and
check that the paths and cookie domains it reports match what you expect before
letting it write anything. [`docs/scope.md`](docs/scope.md) works through how far the
reasoning generalizes and where the evidence runs out.

## What the problem is

The login chain is `vpn.coeit.osu.edu` (Ivanti Connect Secure) → SAML →
`webauth.service.ohio-state.edu` (Shibboleth IdP) → Duo Universal Prompt.

When it fails, it stops on Shibboleth's error page:

> **Unable to complete request** … You may be seeing this page because you waited an
> extended period of time to submit the login form. After a set period, the server
> abandons your original request to save resources, and you'll need to return where
> you started from and try and login again.

The page blames "you were too slow" and iCloud Private Relay — **neither is the real
cause**.

The real cause: Ivanti's embedded browser has only **one global store, shared by
every login attempt**, and there is no way to clear it from the UI. Leftovers from a
previous login that didn't finish cleanly are carried verbatim into the next login;
Shibboleth receives a session it no longer recognizes and spits out the page above.

See [`docs/root-cause.md`](docs/root-cause.md) for details.

## How it surfaces

In rough order of how often each triggers it:

1. **Clicking Disconnect only, without quitting the app.** What actually cleans up
   the IdP-side state is SAML Single Logout, not Disconnect. If SLO doesn't run, the
   leftovers stay.
2. **The app being hard-killed** (force quit / system-update reboot / crash). A
   cookie write gets interrupted halfway, leaving torn state on disk. This machine
   still has a 2026-03-04 remnant.
3. **Abandoning login partway** (Duo push not tapped, network drops, window closed
   outright). This round's session leftovers sit in the store waiting for the next
   round.
4. **Accumulation from never cleaning up.** This machine's HTTP cache holds 13
   one-time SSO callback URLs, the oldest from 7 months ago.

The common thread: none of these produce any visible feedback — you only slam into
it on the next login.

## Usage

```sh
./ivanti-amnesia status      # read-only diagnostics, safe to run anytime
./ivanti-amnesia capture     # gather forensics while stuck (best while the error page is still on screen)
./ivanti-amnesia fix         # repair: quit the app gracefully -> clear leftovers -> relaunch
./ivanti-amnesia restore latest   # roll back
```

> [!CAUTION]
> `capture` writes **unredacted** authentication material to disk on purpose -- the
> live cookie jar with values in the clear, the preferences, and the HTTP cache full
> of one-time SSO callback URLs. `fix` stashes the same thing into its backups.
> Both live in `0700` directories with `0600` files, and `capture` drops a
> `READ-ME-FIRST.txt` into each bundle, but the contents are still live credentials:
> **never attach one to a bug report, issue, ticket or email.** To share findings,
> quote `status` instead -- it redacts every cookie value to a length and an
> 8-character digest.

`fix` has two levels:

| | `--level session` (default) | `--level full` |
|---|---|---|
| SSO / gateway cookie | cleared | cleared |
| Duo device trust | **kept** | cleared |
| HTTP cache | cleared | cleared |
| WebKit data, alt-svc | kept | cleared |
| cost | none | one extra Duo push |

Try `session` first, then `full` if that isn't enough. Add `--dry-run` to preview
without writing anything.

**`fix` refuses to run while the VPN is connected** (unless `--force`), so it won't
interrupt a session in use.

### Prevention

When you're done, Disconnect first, **then fully quit the app with ⌘Q from the menu
bar**. That takes the in-memory session cookie with it — the easiest step to miss and
the most effective one.

## File safety

- Refuses to run as root
- Path allowlist: after `realpath`, a path must live under `~/Library/`, contain the
  bundle id, and not contain `Safari`/`Keychains`/`Containers`. Verified to reject
  Safari cookies, `connstore.dat` (VPN connection config), `~/Documents`, and other
  apps' preferences
- Always backs up to `~/Library/Application Support/ivanti-amnesia/backups/` (0700)
  before deleting; `restore` can roll back — tested that all files are byte-for-byte
  identical after `fix` -> `restore`
- `--dry-run` writes nothing at all
- Never sends SIGKILL (hard-killing is one of the causes of the problem in the first
  place)
- Cookie values are never printed — only their length and an 8-character digest are
  shown. The exception is deliberate: `capture` and `fix`'s backups copy the raw
  cookie jar to disk, under `0700` directories with `0600` files (see the caution
  above)
- The cookie jar is edited in place, as surgery: it runs a byte-level round-trip
  self-check before rewriting and refuses if that check fails

Never touches Safari data, the keychain, `/Library/Application Support/Pulse Secure/`,
or the tunnel daemon running as root.

## Docs

- [`docs/root-cause.md`](docs/root-cause.md) — technical root cause, forensic
  evidence, the binarycookies format
- [`docs/scope.md`](docs/scope.md) — is this an Ivanti-wide problem or specific to
  OSU?

## Dependencies

macOS + the system's built-in python3. No third-party dependencies.
