# vlerv

Claude Code plugin for [Vlervtifacts](https://github.com/contract-hero/vlerv). It
bundles two things:

1. The **`vlerv` MCP server**, which sends local files straight to your paired
   Vlervtifacts devices over a direct, end-to-end encrypted peer-to-peer link.
   Nothing is uploaded to a server.
2. A **SessionStart hook**, which offers device pairing the first time you open
   a Claude Code Remote Control session, so your phone can receive files without
   you having to remember to set it up.

## Install

```
claude plugin marketplace add contract-hero/plugin-marketplace
claude plugin install vlerv@contract-hero
```

The server binary is downloaded on first use from this repository's releases,
checked against the SHA-256 pinned in `bin/manifest.json`, and cached under
`~/.cache/vlerv-plugin/`. A download that fails the checksum is deleted and
never executed.

> If you already added `vlerv` by hand with `claude mcp add`, remove that entry
> first with `claude mcp remove vlerv`. Two servers with the same name is one
> too many.

## Tools

| Tool | Use it for |
|---|---|
| `send_to_device` | A named device: "send this to my phone". Needs the device to grant this server the `control` scope on its own side. |
| `beam_artifact` | A shareable link, or a recipient that is not paired. |
| `list_devices` | Which devices are paired, and what scope each granted. |
| `pair_device`, `pair_status`, `confirm_pairing` | Pairing a new device. |
| `server_status`, `stop_beam` | Diagnosing a failed send, and retiring a link. |

Pairing always needs a person: six words appear on both screens and the human
compares them before `confirm_pairing` runs.

## Send boundary

`VLERV_MCP_ROOTS` is a colon-separated list of directories the server may send
files from. A path outside every root is refused. The plugin leaves it unset, so
it defaults to the directory Claude Code launched the server in. Widen it only
on purpose, in your own MCP settings.

## The pairing hook

`hooks/offer-vlerv-pairing.sh` runs on SessionStart and stays completely silent
unless every one of these is true:

| Condition | Why |
|---|---|
| `CLAUDE_CODE_ENVIRONMENT_KIND=bridge` | `claude rc` sets this in every session it spawns for a phone, before the process starts. A plain local session leaves it unset. |
| `CLAUDE_CODE_REMOTE_SESSION_ID` unset | Drops cloud sessions, which run on Anthropic hardware and cannot reach your machine's files. |
| `~/.claude/.vlerv-paired` absent | Written once a device is confirmed paired. |
| `~/.claude/.vlerv-no-offer` absent | Written if you decline the offer. |

When it does fire, it injects one note asking Claude to call `list_devices` and
offer pairing only when no device is paired. The hook never reads Vlerv state
files, so a change to Vlerv's on-disk format cannot break it.

### Known gap

A session that starts local and turns on Remote Control later with
`/remote-control` is **not** detected. The bridge attaches after SessionStart has
already run, so no SessionStart hook can see it. Sessions started from the phone
against a running `claude rc` are the covered path.

### Re-arm or silence it

```
rm    ~/.claude/.vlerv-paired      # offer pairing again
touch ~/.claude/.vlerv-no-offer    # never offer again
```

## Binary resolution

The launcher tries these in order, so a maintainer never downloads and a user
never builds:

| Order | Source |
|---|---|
| 1 | `$VLERV_MCP_BIN` — explicit override |
| 2 | `$VLERV_SOURCE_REPO/target/**/{release,debug}/vlerv-mcp` — a maintainer's checkout of the private source repo |
| 3 | `~/.cache/vlerv-plugin/` — a verified earlier download |
| 4 | `vlerv-mcp` on `$PATH` |
| 5 | Download from this repo's releases, verify, cache |

Step 2 searches the plain `target/` and the rust target-triple `target/<triple>/`
directories, and runs the **newest** build it finds. `publish-release.sh` builds
with `--target`, so the triple directory is usually the fresh one and the plain
`target/release/` is an older leftover.

Step 2 outranks the cache on purpose. The cache key holds the version pinned in
`bin/manifest.json`, not the build date, so a maintainer who edits the server
and rebuilds keeps re-running the same cached download until the pin moves.

## Releasing (maintainers)

The server source is private, so releases are cut from a machine that has both
checkouts:

```
./scripts/publish-release.sh 0.1.0 ~/workspace/vlerv            # build + pin, no upload
./scripts/publish-release.sh 0.1.0 ~/workspace/vlerv --publish  # upload the release
```

Then commit the updated `bin/manifest.json`, which is what points the launcher at
the new version.

## License

MIT
