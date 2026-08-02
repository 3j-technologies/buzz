# Buzz @ 3J — Onboarding

This repo is 3J's fork of [block/buzz](https://github.com/block/buzz): a
team-chat app built on the Nostr protocol, where humans and AI agents each
get their own cryptographic identity and share the same rooms. We run our
**own relay** rather than any relay operated by Block — all 3J Buzz traffic
stays on our infrastructure.

If you're reading this because someone mentioned "the relay" and you had no
idea it existed: this doc is for you.

## Our relay

- **URL:** `wss://buzz.3jtech.app`
- Live and healthy (verified 2026-08-02). It advertises NIP-42 auth and iOS
  push support over APNs, so it's a real, functioning relay — not a stub.
- Runs on winserver alongside Postgres, Redis, MinIO, and a Cloudflare
  tunnel. Container: `buzz-prod-relay-1`.

## How to get on it today

**⚠️ Honest status: there is currently no easy, ready-to-use client for a
non-technical teammate.** Both routes below have real gaps — read them
before you spend time on either.

### Web client — `https://buzz-web.3jtech.app`

This is **not a usable chat client yet**. It loads (HTTP 200) but has **no
sign-in / npub / nsec surface at all** — it's a repo browser, not a login
screen. There is nothing to click to get into a conversation. Don't spend
time looking for a "sign in" button; it doesn't exist yet on this build.

### Desktop (Tauri app)

The desktop app *can* be pointed at our relay — this is confirmed by
reading the source, not by producing a working build:

- At runtime, the app reads the `BUZZ_RELAY_URL` environment variable
  first (`desktop/src-tauri/src/relay.rs`).
- If that's unset, it falls back to a value baked in at **build time**
  from the same `BUZZ_RELAY_URL` variable (`desktop/src-tauri/build.rs`).
- If neither is set, it defaults to `ws://localhost:3000`.
- **Precedence, confirmed by reading the code:** a runtime env var always
  wins over whatever was baked in at build time. Setting `BUZZ_RELAY_URL`
  when you launch the app overrides a build that was compiled pointing
  somewhere else.

**There is no distributable desktop build for the team yet.** The repo's
canary/release workflows (`signed-macos-canary.yml`, `windows-canary.yml`,
`linux-canary.yml`, `release.yml`) all contain the same guard:

```yaml
if: github.repository == 'block/buzz'
```

That means those workflows **cannot produce a build on our fork at all**,
even via manual `workflow_dispatch` — the job condition fails immediately,
before it would even reach the (Block-only) signing secrets it also
depends on. So today, getting a desktop build means compiling from source
yourself (`just desktop-build` after setting `BUZZ_RELAY_URL`), and the
result would be **unsigned** — expect Gatekeeper/SmartScreen warnings on
macOS/Windows, since we have no code-signing pipeline wired for this app.
**We have not actually produced or tested a local build against our relay
— this is unverified.** If you try it and it works, please update this
doc.

### iOS — the App Store app

Block publishes an official app, **"Buzz — chat with your hive"**, on the
iOS App Store (confirmed live, App ID `6779728271`). This is a real,
polished client — much further along than our web or desktop story.

The relay is **not hardcoded** in the mobile app: on first launch it goes
through a **pairing flow** (QR code / pairing code, NIP-42 handshake) where
the relay URL comes from whatever is being paired against — normally an
already-running **desktop** client. In other words, the App Store app could
point at `buzz.3jtech.app`, but only by pairing against a desktop instance
that's already configured for our relay.

**Given the desktop-build gap above, this path is currently blocked in
practice** — we don't yet have a way to hand someone a desktop build to
pair from. This is a real chicken-and-egg gap, not a "just try it" item.

## Identity: what's an npub/nsec?

Buzz accounts are Nostr keypairs, not usernames/passwords:

- **`npub`** — your public identity. Safe to share; this is how others
  find/mention you.
- **`nsec`** — your **private key**. It is the only thing that proves
  you're you. **Never paste it into a chat message, a form, a Slack
  message, or anywhere outside the Buzz app itself.** Anyone who has your
  `nsec` can act as you, permanently, with no reset process (there's no
  "forgot password").

How you get one: the desktop app has a real onboarding flow
(`desktop/src/features/onboarding/`) that creates a fresh keypair for you
locally the first time you set up a community. We have not walked a
teammate through this against our relay yet, so treat the exact click-path
as unverified until someone reports back.

## What the relay does and doesn't guarantee today

From the relay's own NIP-11 metadata (`auth_required: true`,
`restricted_writes: true`) and the team's review of the relay code as of
2026-08-02: **any authenticated user can currently read**, i.e. there is no
per-channel/per-room read gating yet. Don't post anything in a 3J Buzz
channel that you wouldn't want every other authenticated user on the relay
to see. **This has not been independently re-verified against a live
account** (we don't have a working login path yet — see above) — it's
based on reading the relay's access-control code, not on testing it as two
different users. If you find this is wrong once real accounts exist,
correct this doc.

## Where to ask for help

Ask in the usual 3J channel where this doc was shared, or reach whoever is
currently driving the Buzz relay/infra work (check the tracker project
`INFRA` for the current ticket — search for "Buzz"). Please update this
doc with anything you verify that's marked unverified above.
