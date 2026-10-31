# vulnwatch

vulnwatch finds vulnerable dependencies in your GitHub repos and fixes them on its own. A sandboxed coding agent makes the fix, a second model reviews it, and it merges once checks pass. No human is in the loop.

Each pass takes a few minutes:

1. **Watch.** It lists every repo you can see that was pushed recently and reads its lockfiles. Then it checks the exact installed versions against [cve.now](https://cve.now), a free merged feed of the CVE List, GitHub advisories, OSV, NVD, EUVD, CISA KEV and EPSS. It also reads GitHub's Dependabot alerts and switches them on for your repos. New repos and new dependencies are picked up on the next pass after a push.
2. **Notify.** It sends one line per repo with new findings, for example `Vulnerable: my-site uses vite 8.2.3 (CVE-2026-112942, high) +5 more. Fixing it now.`
3. **Fix.** A Codex agent works on a plain copy of the repo inside Codex's OS sandbox, with every read and write under `$HOME` denied. It records install, build and test results before its change and again after it.
4. **Review.** A second model (GPT-6.1 Sol by default) reviews the PR in a fresh session that sees only the diff and the PR description. The author gets up to 3 rounds to fix or rebut its findings. The review must end in `AGREE` on the exact pushed commit.
5. **Merge.** The merge is pinned to that commit and happens only when all of these hold:
   - the reviewer agreed;
   - no lockfile pulls from a host outside the known registries;
   - every check on the commit passed, including every workflow that runs on pull requests.

   If no workflow runs on pull requests, only dependency-file-only diffs may merge. If checks on the default branch fail after a merge, it opens a revert, has it reviewed, and merges the revert.
6. **Report.** It sends one line per outcome, such as `Fixed and merged my-site: vite, devalue. https://github.com/me/my-site/pull/18`.

Its first pass over 71 active repos found 608 vulnerable dependency matches. On one Next.js repo, cve.now matched 15 issues where Dependabot had 11 alerts. In its first end-to-end run, it fixed 4 packages in one repo, the reviewer agreed in the first round, and it merged with no human involved. Where the only fix was a major upgrade it couldn't verify, it reported "couldn't fix" and changed nothing.

## Safety model

The agent that writes the fix is treated as untrusted. Repo files, package metadata and advisory text can all contain prompt injection.

- **No credentials for the agent.** It runs under a Codex permission profile that denies `$HOME`, so it can't read `~/.config/gh`, `~/.git-credentials`, SSH keys, the kill switches or this script. Its network stays on so package managers work. Its working copy has no `.git` folder.
- **A trusted side does the git work.** This script clones the repo into `state_dir`, where the sandbox can't reach. It moves the agent's copy there before reading it, mirrors the files over without `.git`, and commits with hooks off. Then it pushes, opens the PR, and merges with `--match-head-commit`.
- **An independent review.** The reviewer runs in an empty folder in read-only mode, on a diff taken from the trusted checkout. Very large lockfile diffs drop only unchanged context and integrity hashes. Every changed package line stays.
- **CI on the exact commit.** Checks are read from the commit's check runs, statuses and Actions runs, not from the PR.
- **Kill switches.** `touch <state_dir>/DISABLE_AUTOFIX` switches to alert-only, and running fixers stop before their next push or merge. `touch <state_dir>/DISABLE_AUTOMERGE` keeps opening PRs but never merges.

Known limits:

- In repos whose CI runs only on pushes to the default branch, a dependency-only fix merges on the agent's sandboxed before-and-after results plus the review. The post-merge revert is the backstop.
- Browser tests (Playwright, Chromium) can't launch inside the Codex sandbox on macOS. Fixes in those repos tend to end as "couldn't fix" rather than merge.
- Forks are skipped by default because their dependencies usually track upstream.
- cve.now keeps each uploaded lockfile's package names and versions so it can serve that repo's feed. The lockfile file itself is deleted after parsing.

## Requirements

- Python 3.11 or newer, `git`, `rsync`
- [GitHub CLI](https://cli.github.com) signed in with `repo` scope (`gh auth login`)
- [Codex CLI](https://github.com/openai/codex) signed in, with a version that supports permission profiles (`default_permissions`)
- The package managers your repos use (npm, pnpm, yarn, bun, uv, pip, cargo, go, bundler)
- macOS, or Linux with Codex's sandbox available

## Setup

```sh
git clone https://github.com/hunterbohm/vulnwatch ~/vulnwatch
mkdir -p ~/.config/vulnwatch
cp ~/vulnwatch/config.example.toml ~/.config/vulnwatch/config.toml
# set github.owners, and notify.command if you want texts or Slack messages
python3 ~/vulnwatch/vulnwatch.py --dry-run
```

The dry run lists every finding and every fixer it would start. It changes nothing.

Then run it every 5 minutes. With cron:

```cron
*/5 * * * * /usr/bin/env python3 $HOME/vulnwatch/vulnwatch.py >> $HOME/.local/state/vulnwatch/notify.log 2>&1
```

The first pass records what already exists without notifying you, then starts fixers for repos that have findings, 2 at a time. After that, you hear only about new findings and outcomes.

## Watching the fixes

By default each fixer runs in the background and logs to `<state_dir>/logs/`. To watch fixes instead, set `fix.launch_command`, for example one tmux window per fix:

```toml
[fix]
launch_command = "tmux new-window -d -n fix-{name} {fix_cmd} && echo fix-{name}"
done_command = "tmux kill-window -t {handle} 2>/dev/null || true"
```

The same hooks work for an agent app that can start threads from the command line. The thread runs `{fix_cmd}`, and `done_command` closes it when the fix merges.

## Machines

vulnwatch covers code. For servers and laptops, set `hosts.on_kev_command` to your own patch script. It runs whenever CISA adds an actively exploited vulnerability, and the new ids are in `$VULNWATCH_NEW_KEV`.

## Tests

```sh
python3 -m unittest discover -s tests
```

## License

MIT
