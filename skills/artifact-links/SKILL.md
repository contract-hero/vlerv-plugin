---
name: artifact-links
description: >-
  Write a vlerv:// link to a local file that opens on ANY of the user's paired
  devices, not just this Mac. Use whenever you hand the user a link to a file
  you produced — a report, an audit, an explainer, any deliverable — or when
  they ask to share, send or open a file on their phone, tablet or other Mac.
  Also use when a link failed on a device with "canonicalize failed", "No such
  file or directory", or a path that plainly exists on the Mac. Short version:
  never hand-write vlerv://open?path=…, always mint it with share_link.
---

# Writing vlerv:// links that work everywhere

## The mistake this prevents

A hand-written link names a path and nothing else:

```
vlerv://open?path=%2FUsers%2Fyou%2Fworkspace%2Freport.html      ← BROKEN on the phone
```

It opens on the Mac and fails on every other device, because the path names a
file on a machine **the link never identifies**. The phone looks on itself,
does not find it, and shows:

```
Deep link rejected: canonicalize failed at "/Users/you/workspace/report.html":
No such file or directory (os error 2)
```

Nothing is wrong with the file. The link was incomplete.

## The fix: mint it, never type it

Call `share_link` with the absolute path. It returns a link carrying
`from=<this Mac's node id>`:

```
vlerv://open?path=%2FUsers%2Fyou%2Fworkspace%2Freport.html&from=e35eb3e489…
```

Give the user **that** string, verbatim.

## One link, both places

There is no separate "local version" to also produce:

| Opened on | Behaviour |
|---|---|
| The Mac that minted it | `from` matches this install, so it is dropped — an ordinary **local open** |
| A paired device | `from` names a peer — the device **pulls** the file from that Mac |
| An unpaired device | "not paired with the device this link comes from", pointing at Settings → Devices |

A minted link is a superset of a hand-written one, so there is no judgement
call. **Always mint.**

## Two things to tell the user when they matter

**Vlervtifacts must be running on the Mac when the link is opened.** The pull
is peer-to-peer with nothing uploaded and nothing queued. If the Mac is closed
the other device says "Device unreachable / Try again", which is accurate.

**iOS does not make a custom scheme tappable in plain text.** A `vlerv://`
link pasted into Notes or Messages stays inert. What works: the macOS share
sheet (Share ▾ → *Share link…*), a QR code, or pasting into the app's address
bar.

## If the device is not paired yet

`share_link` reaches only devices the user has already paired. To pair one:
`pair_device` mints a `vlerv://pair?ticket=…` link, the user opens it on the
device, then six words appear on both screens. **Show the words from
`pair_status` verbatim and ask the user to compare them** — that comparison is
the only thing standing between them and a machine in the middle. Finish with
`confirm_pairing { accept: true }`, and never confirm on their behalf.

## Someone who is not the user

`share_link` is for the user's own devices. For anyone else use
`beam_artifact`, which stages the bytes and mints a `vlerv://receive?ticket=…`
link anyone holding it can fetch while the app runs. Different trust shape: a
beam link is a bearer capability for one file. `beam_artifact` is confined to
`VLERV_MCP_ROOTS`; `share_link` is not, because it reaches only the user's own
machines.

## Before you send a link

1. Did `share_link` produce it? If you typed `vlerv://open?path=` yourself, stop.
2. Is it for the user's own device? If not, `beam_artifact`.
3. Will the Mac be running when they open it? If not, say so.
