# Upgrading our fork

This is the fork `udiNur/tuios`. We run upstream `Gaurav-Gosain/tuios` `main`
plus a small stack of our own patches, built locally and installed into
`/opt/homebrew/bin`. This page is the procedure for moving to a newer upstream.

Remotes in `~/Projects/tuios`: `origin` = upstream, `fork` = ours.

## Our patches

Cherry-picked on top of every upstream base (as of `integ/6a8c093d0-local`):

| Patch | What it does |
|---|---|
| `feat(list-clients): report last_input_at` | `list-clients` reports when the person last gave input at each client |
| `fix(harness): a Codex option that wraps over many lines keeps its menu` | the Codex option menu survives wrapped options |
| `fix(session): a client moved between sessions cannot carry one session's state into another` | window size and state do not leak across a session switch |

Drop a patch once upstream fixes the same thing.

## 1. Build on the new upstream base

```sh
cd ~/Projects/tuios && git fetch origin
BASE=$(git rev-parse --short=9 origin/main)            # the upstream commit to move to
git worktree add -b integ/$BASE-local ~/Projects/tuios-wt-$BASE origin/main
cd ~/Projects/tuios-wt-$BASE
git cherry-pick <last_input_at> <codex-option> <session-state>   # the 3 patches above
just build                                              # .dev/bin/tuios and tuios-web, version dev+<commit>
.dev/bin/tuios --version
```

Upstream now generates `docs/protocol-verbs.md` from the verb registry, and the
`last_input_at` patch changes the `list-clients` text. If the doc test fails,
write it again inside that patch:

```sh
go test ./internal/session -run TestProtocolVerbsDocMatchesRegistry -update
```

## 2. Test

```sh
just test                  # unit; a few timing tests may fail under heavy load, rerun them alone
just e2e                   # e2e
just sandbox               # smoke test on a throwaway daemon (own socket and state)
```

Known e2e failures (18.11.2026, base `6a8c093d0`), both in the cross-machine
link tests: `TestMailCrossesTheLink`, `TestWindowSizeLinkClientsUseTheirOwnEdges`.
Upstream's own e2e was red at the time.

In the sandbox, check at least: window size across a session switch, the TML
popup, the queue, agent states, mail between sessions, `last_input_at`, an old
client against the new daemon, and a live `daemon upgrade` from the installed
binary to the new one (same pid, panes stay).

Stop the scratch daemons afterwards: `just sandbox-stop`.

## 3. Install and upgrade the live daemon

Run this in a **plain Ghostty tab, not inside a tuios pane**. Inside a pane the
upgrade asks for confirmation in a dialog.

**Never `cp` over the running `/opt/homebrew/bin/tuios`.** macOS keeps the old
file's code signature for that inode and kills every run of the changed file
(`zsh: killed`, exit 137). Copy to a temporary name in the same folder, then
`mv` it over: that is a new inode with its own valid signature.

```sh
OLD=$(tuios --version | sed -E 's/.*dev\+([0-9a-f]+).*/\1/')
NEW=~/Projects/tuios-wt-$BASE/.dev/bin
B=/opt/homebrew/bin

# backup
cp -p $B/tuios     $B/tuios.bak-$OLD
cp -p $B/tuios-web $B/tuios-web.bak-$OLD

# install: temp name + mv, never cp over the file
cp -p $NEW/tuios     $B/.tuios.new     && mv -f $B/.tuios.new     $B/tuios
cp -p $NEW/tuios-web $B/.tuios-web.new && mv -f $B/.tuios-web.new $B/tuios-web
tuios --version

# live upgrade: same pid, every pane and program stays
tuios daemon upgrade
```

Before upgrading: no agent should have queued text (`list-agents`, `queued > 0`),
because queued text does not survive a live upgrade.

If the upgrade does not report `upgraded`, nothing changed. Do not add
`--or-restart`; read the reason first.

## 4. Verify

```sh
PID=$(pgrep -f "^/opt/homebrew/bin/tuios daemon" | head -1)   # the live daemon, not test daemons from a worktree
lsof -p $PID | awk '$4=="txt" && /tuios/ {print $7, $9}'    # size must equal the new binary's size
ls -l $NEW/tuios
tuios ls
```

The daemon runs from a handoff copy under
`~/Library/Application Support/tuios/sessions/handoff/`, so compare sizes, not paths.

Connected clients are still processes of the old binary. Detach and attach
each one again (`tuios attach <session>`), including the phone.

## 5. Roll back

The same way, from the backup:

```sh
cp -p $B/tuios.bak-$OLD $B/.tuios.new && mv -f $B/.tuios.new $B/tuios
cp -p $B/tuios-web.bak-$OLD $B/.tuios-web.new && mv -f $B/.tuios-web.new $B/tuios-web
tuios daemon upgrade
```

## 6. Clean up

- Stop leftover test daemons started from the worktree:
  `pgrep -fl "tuios-wt-.*/.dev/bin/tuios daemon"`, then `just sandbox-stop` in that worktree.
- Keep the last two `tuios.bak-*` files; older ones can go.
