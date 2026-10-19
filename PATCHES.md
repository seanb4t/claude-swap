# Fork patches

This fork is [realiti4/claude-swap](https://github.com/realiti4/claude-swap)
`main` with one commit for each patch on top. `git log upstream/main..main`
lists them. Each upstream update rebases the series onto the new upstream
`main`. The fork never merges upstream.

The patches are in series order. Patches that can go upstream come first, so
that each one can be sent from a branch cut from upstream `main`.

## 1. fix(usage): keep the live shared fields when rewriting the active store

- **Why:** a usage pass on the active account (`cswap list`, `status`, `auto`)
  can restore the slot backup into the live store, or refresh the backup's
  grant. It wrote the backup's copy of `mcpOAuth` and the other shared fields
  over the live ones. An MCP token that rotated after the backup then goes
  back to an older generation, and the MCP server asks for a new login.
- **Upstream:** issue [#479](https://github.com/realiti4/claude-swap/issues/479),
  PR [#481](https://github.com/realiti4/claude-swap/pull/481). The same bug
  class as #135, fixed for switches in #145.
- **Plan:** the fork carries the PR commit. Drop it when upstream merges it.

## 2. feat(settings): updates.check turns off the daily PyPI update check

- **Why:** this install is pinned to a fork commit and upgraded by a dotfiles
  manager, so the "newer version" hint is noise and the request to PyPI is
  not necessary. `cswap config set updates.check false` turns it off.
- **Upstream:** issue [#480](https://github.com/realiti4/claude-swap/issues/480),
  PR [#482](https://github.com/realiti4/claude-swap/pull/482). It replaces the
  fork's earlier `hardening.noUpdateCheck` and `CLAUDE_SWAP_NO_UPDATE_CHECK`.
- **Plan:** the fork carries the PR commit. Drop it when upstream merges it.
  If upstream takes a different shape, adopt that shape and drop this patch.

## 3. feat(settings): hardening options in settings.json

- **Why:** patches 4 and 5 must hold for every invocation: any shell, an
  agent, a LaunchAgent. This patch adds a `hardening` section
  (`keychainOnly`, `noActiveRefresh`) that `cswap config` lists and sets, with
  a `CLAUDE_SWAP_*` override for each. A file or value that cannot be read
  turns an option on, because each option is the stricter behavior.
- **Upstream:** fork only. The maintainer prefers to mirror Claude Code and
  avoid user options (#64, #66), and upstream reads a bad setting as its
  default. Failing closed contradicts that.
- **Plan:** shrinks or goes away as patches 4 and 5 go upstream.

## 4. feat(usage): hardening.noActiveRefresh leaves the active token to Claude Code

- **Why:** Claude Code's refresh tokens rotate, so each one works once. When
  cswap refreshes the active account's token during a usage pass, every
  running Claude Code session keeps the old refresh token. Their next refresh
  fails with `invalid_grant`, and they log out. The patch also stops the
  refresh gate from spending a grant that is, or can be, the live one, for
  example when a pass reads `~/.claude.json` while it is rewritten. A security
  review of upstream on 2026-09-18 found the risk. Upstream reports that match
  it: #433, #443.
- **Upstream:** not proposed. The maintainer built active refresh on purpose
  (#70, #167).
- **Plan:** propose the gate check as an always-on fix first (issue citing
  #433 and #443), separate from the option. Then propose the option as a
  setting that is off by default.

## 5. feat(credentials): hardening.keychainOnly keeps every credential write in the Keychain

- **Why:** on macOS, a credential write goes to the Keychain or fails. No
  `~/.claude/.credentials.json`, no `.enc` backups, no unclaimed stash, no
  `primaryApiKey` fallback. A write that `security` reports as failed counts
  as done when a read-back returns the secret.
- **Upstream:** fork only. Upstream mirrors Claude Code's file fallback on
  purpose: #66 added the automatic fallback, and #21, #64, #280 and #326 were
  closed.
- **Plan:** the read-back check can go upstream as a robustness fix (related:
  #254, #441). The rest stays in the fork.

## 6. docs(fork): list the fork's patches

This file. It never goes upstream.

## Candidates

Changes a review found that are not patches yet. Each one can go upstream as
it stands.

- **Lock the account index on every write.** `cswap threshold`, `alias` and
  `disable` rewrite `sequence.json` without the switch lock. If one runs while
  an auto-switch writes the index, the index can point at the previous slot.
  No credential moves. Found by the security review of upstream `9ea59b2` on
  2026-10-19.

## Adding a patch

1. Write it as an upstream change: the smallest change, upstream's
   conventions, a test that fails without it, and no dependency on a
   fork-only patch where that is possible.
2. Make it one commit. The test suite must pass at every commit of the series.
3. Put it in the series: after the patches that can go upstream if it can
   too, otherwise after the fork-only patches and before this file.
4. Add a section here: why it exists, its upstream status, and the plan.
5. If upstream can want it, open an issue, then a PR from a branch cut from
   upstream `main` with the same commit. When upstream merges the PR, the
   next rebase drops the commit from the series.
