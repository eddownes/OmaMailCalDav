# Reinstalling Omamail with Fastmail CalDAV discovery

This fork ([eddownes/OmaMailCalDav](https://github.com/eddownes/OmaMailCalDav)) is
[huacnlee/omamail](https://github.com/huacnlee/omamail) plus the generic CalDAV server
discovery proposed upstream in
[huacnlee/omamail#292](https://github.com/huacnlee/omamail/pull/292).
Use it on a fresh Omarchy install until that PR is merged and released.

**The plugin checkout alone is not enough.** The fix has two halves, and the installer
only gives you one of them. Skipping step 2 below is why discovery "does nothing" on a
new machine.

## Why a fresh install doesn't discover Fastmail calendars

| Half | Where it lives | What a normal install gives you |
| --- | --- | --- |
| The **Discover calendars...** wizard | QML in `ui/` | This fork's version, straight from the checkout. |
| The `calendar.discoverCaldavServer` RPC the wizard calls | Rust in `src/calendar/discovery.rs` | **Upstream's prebuilt backend**, downloaded from upstream's release for the version in `backend-version`. It does not have the RPC. |

So the wizard is there but the backend behind it can't answer. The backend has to be
built from this checkout instead.

Branches:

- `fastmail-caldav`: the fix plus this guide. Install from this one.
- `caldav-server-discovery`: exactly what PR #292 contains. Don't add personal changes
  there, because anything pushed to it shows up in the PR.

The tag `fastmail-caldav-2026-09-25` marks the tested code (this branch adds only this guide on top of it).

## Before you start: create a Fastmail app password

Your normal Fastmail password won't work for CalDAV.

1. In Fastmail, open **Settings → Privacy & Security → Manage app passwords**.
2. Click **New app password**. For access, choose **Calendars (CalDAV)** or a set that
   includes it.
3. Copy the password. You'll paste it into the wizard in step 3 below.

## 1. Install the plugin from this fork

```bash
omarchy plugin add https://github.com/eddownes/OmaMailCalDav.git --enable --yes
git -C ~/.config/omarchy/plugins/omamail fetch origin fastmail-caldav
git -C ~/.config/omarchy/plugins/omamail checkout --detach FETCH_HEAD
```

Open Omamail once and let it finish its own setup (it downloads upstream's backend to
`~/.local/share/omamail/bin/omamail`). Step 2 replaces that binary.

## 2. Build the backend from this checkout

Needs `cargo` (`rust` package on Arch/Omarchy). The build takes about three minutes.

```bash
cd ~/.config/omarchy/plugins/omamail
make install-backend-local   # cargo build --release, then installs into ~/.local/share/omamail/bin/
pkill -x omamail             # the backend is on-demand; it restarts on the new binary
```

`make install-backend-local` ends by printing a status line. It should say
`"state": "ready"` and list `"unreleasedMethods": ["calendar.discoverCaldavServer"]`.
The build goes to `target/`, which is git-ignored, so the checkout stays clean.

## 3. Add your Fastmail calendars

1. Open Omamail, then **Settings → Calendars → Discover calendars...** (next to
   **Add a calendar**).
2. **Server address:** `https://caldav.fastmail.com`.
   **Username:** your full Fastmail address.
   **Password:** the app password from above.
3. Tick the calendars you want and confirm.

## 4. Check that it works

```bash
# 1 = the installed backend has the discovery RPC; 0 = it is upstream's binary, redo step 2
strings -a ~/.local/share/omamail/bin/omamail | grep -c discoverCaldavServer

# Fastmail's well-known redirect, which discovery follows: expect 301 -> .../dav/calendars
curl -sS -o /dev/null -w '%{http_code} -> %{redirect_url}\n' -X PROPFIND https://caldav.fastmail.com/.well-known/caldav
```

## Troubleshooting

| Symptom | Cause / fix |
| --- | --- |
| **Discover calendars...** is there but finds nothing or errors immediately | The backend is upstream's prebuilt binary. Run the first check in step 4; if it prints `0`, redo step 2. |
| It worked, then stopped after an update | A plugin update or backend reinstall put upstream's binary back. Redo step 2. |
| No **Discover calendars...** button at all | The checkout isn't this fork's branch. Check `git -C ~/.config/omarchy/plugins/omamail remote -v` and `log -1`. |
| "Local backend does not match Cargo package version." | `backend-version` and `Cargo.toml` disagree, usually after a partial update. Check out this branch cleanly and rebuild. |
| Credentials rejected | You used your account password or an app password without CalDAV access. Create a new one as described above. |
| Wizard still fails right after step 2 | The old backend process or shell is still running: `pkill -x omamail`, and if needed `omarchy restart shell`. |

## Once PR #292 is merged upstream

Switch back to the official plugin; its released backend will then include the RPC.
Your accounts and calendars in `~/.config/omamail/` are kept.

```bash
omarchy plugin remove omamail --yes
omarchy plugin add https://github.com/huacnlee/omamail.git --enable --yes
```
