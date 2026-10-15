# iOS Signing Window Data

To my knowledge, this is the only public dataset that tracks how long Apple keeps
signing each iOS / iPadOS / watchOS build: when a signing window opens, when it
closes, and how that compares to the average.

Sites like ipsw.me show what is signed *right now*. They don't keep a history of
how long each window lasted, or whether Apple closed one faster or slower than usual.
This repo does.

The data updates automatically. A tracker checks Apple every 5 minutes, and this
repo gets a commit once a day if anything changed (a new release, a window opening
or closing, and so on).

**Live alerts:** follow [@zimzam___](https://x.com/zimzam___) on X for new releases,
signing-window starts and closes as they happen.

## Why track this?

After a new iOS release, Apple keeps signing the previous version for a while, then
stops. **A window that closes unusually fast can mean Apple is hurrying to shut a
door,** often because the new release fixes a serious, possibly exploited,
vulnerability.

If you're **marinating**, a fast close on the version you're on
is a hint there may be something worth digging into. Check what the replacement
build fixed. That could let you re-discover the CVE and look for bypasses of the
fix, or point to something Apple fixed internally without disclosing it. That last
part is speculative.

Each closed window is compared against the average for its OS (`delta_days`;
negative = faster than usual). Treat it as a hint, not proof: Apple doesn't say why
it stops signing, and windows can be short for routine reasons.

## Files

| File | What it is |
|------|------------|
| `signing.json` | One row per OS build: signing window start, close, duration, and how it compares to the running average |
| `seen_updates.json` | Every iOS / iPadOS / watchOS release seen (a few early tvOS rows are from before tvOS was dropped), with its release date, when the tracker first saw it, and its `kind` (see below) |
| `state.db` | The same data as a SQLite database |

The git history is the timeline: `git log -p signing.json` shows each window opening
and closing as it happened.

## How signing is checked

- Signing status comes **straight from Apple's TSS server** via
  [`tsschecker`](https://github.com/tihmstar/tsschecker), not from third-party signing sites.
- A signing window is per build and is the same on every device running that build,
  so each OS is checked on one reliable anchor device:

  | OS | Anchor device |
  |----|---------------|
  | iOS | iPhone SE 2 (`iPhone12,8`) |
  | iPadOS | iPad Air 4 (`iPad13,1`) |
  | watchOS | Apple Watch Series 9 (`Watch7,1`) |

- **Window = how long Apple keeps signing a build after a newer version replaces it**
  (the downgrade window). The clock starts when the newer version is released.
- A build only counts as closed after **two NOT-signed answers in a row**. The close
  time is the first of the two. Network or tool errors count as "unknown" and never
  change the state.

## `seen_updates.json` kinds

| `kind` | Meaning |
|--------|---------|
| `major` / `minor` / `patch` | A public release: `27.0` / `27.1` / `27.0.1` |
| `upcoming` | A pre-listed build: firmware AppleDB lists for hardware that hasn't launched yet (e.g. iOS 27.1 for the iPhone Duo, `iPhone19,4`), or a build dated in the future. `released` is AppleDB's date for the build, not the hardware launch |

## `signing.json` fields

| Field | Meaning |
|-------|---------|
| `os`, `version`, `build`, `device` | The build and the anchor device it was checked on |
| `released` | Release date of this build |
| `superseded_at` / `superseded_by` | When the replacing build came out, and its build number (the window start) |
| `superseded_src` | Where `superseded_at` came from: `ipsw.me` (exact timestamp), `observed` (first seen by the tracker), or `date` (day only, from AppleDB) |
| `first_signed_at` | When the tracker first saw this build signed |
| `unsigned_seen_at` | First NOT-signed reading (a close is confirmed on the next reading) |
| `closed_at` | When signing stopped |
| `is_signed` | `1` = signed at the last check, `0` = not signed |
| `duration_days` | How long the window lasted |
| `avg_at_close` | Average window length for that OS at the time this one closed |
| `delta_days` | `duration_days − avg_at_close`. **Negative means Apple closed it faster than usual** |
| `predicted_close_at` | Predicted close time, based on the running average |
| `*_alerted` | Internal flags for which alerts have already been sent |

Times are ISO 8601. Precision depends on `superseded_src`: windows that start from
`ipsw.me` or `observed` are accurate to within about 5 minutes. `date` windows are
accurate to the day.

## Coverage

Tracking started in 2026, so the history begins there. Averages only become
meaningful once a few windows have closed for each OS.
