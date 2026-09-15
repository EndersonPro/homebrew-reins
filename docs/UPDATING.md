# Updating reins-hook with Homebrew

This is the whole update path for a Homebrew install. It does not need the
`reins` source tree, a Go toolchain, or any script from the monorepo.

```sh
brew update
brew upgrade reins-hook
reins-hook install
brew services restart reins-hook
```

Four commands, and every one of them is load-bearing. The rest of this page
explains why, and how to prove the update actually landed.

---

## Why each step

**`brew update`** refreshes the tap. Homebrew will not see a new `reins-hook`
release until the `endersonpro/reins` tap is pulled, so `brew upgrade` on its
own can report "already up to date" against a formula that is weeks old.

**`brew upgrade reins-hook`** replaces the binary in the Cellar and relinks
`/opt/homebrew/bin/reins-hook` (`/usr/local/bin` on Intel).

**`reins-hook install`** rewrites the pieces that live outside the binary. The
Claude Code hooks, the OpenCode bootstrap plugin, and the Pi bridge extension
are embedded in the binary and only reach disk through this command. Skip it
and the upgraded gateway talks to integrations from the previous version —
which fails quietly, in whatever way the two versions happen to disagree.

**`brew services restart reins-hook`** is what actually puts the new code in
memory. A running process keeps the image it loaded at start; rewriting the
file on disk changes nothing until the process is restarted. This is the step
people skip, and it is the reason an app still reports the old version after a
successful upgrade.

If you run the gateway by hand instead of through `brew services`, use
`reins-hook restart` in its place.

---

## Verify it landed

Ask the binary, then ask the running gateway. They are different questions and
only the second one matters:

```sh
reins-hook version
curl -s http://127.0.0.1:24543/health
```

`/health` reports the version of the process that is actually serving. If it
disagrees with `reins-hook version`, the restart did not take.

To see what the gateway believes the newest release is:

```sh
curl -s http://127.0.0.1:24543/hook/update
```

```json
{
  "current_version": "0.6.0",
  "latest_version": "0.6.0",
  "update_available": false,
  "release_url": "https://github.com/EndersonPro/homebrew-reins/releases/tag/v0.6.0",
  "checked_at": "2026-09-15T12:50:39Z"
}
```

An empty `latest_version` right after a restart is normal, not a failure. The
check runs in the background with a short random startup delay and then every
six hours; before the first one completes, the gateway has nothing to report.
Give it a few seconds and ask again. It never blocks the request path, so this
route is always a cheap read of the last known good answer.

---

## Two things that will bite you

### A non-Homebrew copy earlier in `PATH`

If you ever installed `reins-hook` from source or with the install script, a
second binary is probably sitting in `~/.local/bin`, which most shells put
ahead of Homebrew. Then `brew upgrade` succeeds, and every command you type by
name still runs the old copy. Homebrew warns about this during `brew info`:

```
The following reins-hook executables are shadowed by other commands earlier in your PATH:
  reins-hook (shadowed by /Users/you/.local/bin/reins-hook)
```

Check which one you are actually running:

```sh
which -a reins-hook
```

If more than one path comes back, remove the copy you are not maintaining —
running two installs of the same tool is not a configuration, it is a bug you
have not hit yet.

### Port 24543 already in use

Only one gateway can bind `127.0.0.1:24543`. If another copy is already
serving, the Homebrew service cannot start, and because the formula sets
`keep_alive true`, launchd retries it forever. `brew services list` shows
`error` and the log fills with the same line:

```
reins-hook: listen on 127.0.0.1:24543: listen tcp 127.0.0.1:24543: bind: address already in use
```

Find the process holding the port and stop it before starting the service:

```sh
lsof -nP -iTCP:24543 -sTCP:LISTEN
reins-hook stop
brew services restart reins-hook
```

The gateway's log lives at `/opt/homebrew/var/log/reins-hook.log`.

---

## When the app still shows an old version

The Reins app reads the version from the gateway's `/health`, so it can only
be as current as the running process. In order:

1. `curl -s http://127.0.0.1:24543/health` — if this is old, the gateway was
   never restarted. Go back to step four.
2. `which -a reins-hook` — if a non-Homebrew copy shadows the tap's, the
   service may be running that one.
3. Restart live Claude Code, OpenCode, and Pi sessions. They register with the
   gateway at startup, and a gateway restart drops every registration; until
   they reconnect the board reports no agents.

Only the `major.minor.patch` triplet is compared, so build suffixes such as
`0.6.0-1-gabc1234` are read as exactly `0.6.0`.
