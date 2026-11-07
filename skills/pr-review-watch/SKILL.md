---
description: One pass of the review watcher; finds open PRs that request your review and runs /tradecraft:pr-review --publish on each in its own headless session. Run it on a timer with /loop.
argument-hint: [clone root dir]...
---

Run one pass of the review watcher. Start it with:

```
/loop 15m /tradecraft:pr-review-watch
```

The clone roots are the directories that hold your local clones; a clone is a direct child of one of them. `$ARGUMENTS` lists them; when it is empty, read them from `~/.config/pr-review-watch/roots`, one directory per line, `~` allowed. When neither gives a root, print `no clone roots: pass directories or write ~/.config/pr-review-watch/roots` and end the pass; do nothing else.

Each review runs in its own `claude -p` process with a fresh context, so this session never holds a review. It sees one summary line per PR. Never use em dashes. Keep each pass short.

State lives in `~/.cache/pr-review-watch/`: the directory `lock` exists while a batch runs and holds the batch's pid in `lock/pid`; `results/<owner>/<repo>/<number>-<head7>.json` and `.err` hold the one attempt at that head.

## 1. Take the lock

`mkdir -p ~/.cache/pr-review-watch && mkdir ~/.cache/pr-review-watch/lock` is the lock: `mkdir` either creates it or fails, with no gap between the check and the claim. When it fails, read `lock/pid`; if `kill -0` on that pid succeeds, print `review batch running` and end the pass; if not, the batch died, so `rm -r lock` and take it again. When `lock/pid` is missing, step 5 has not written it yet: treat the lock as live and end the pass if the `lock` directory is younger than ten minutes (`find ~/.cache/pr-review-watch/lock -maxdepth 0 -mmin +10` prints nothing). Only a pid-less lock older than that is stale. Release the lock with `rm -r lock` at the end of a pass that starts no batch.

## 2. Find requests

```sh
gh search prs --review-requested=@me --state=open --draft=false \
  --updated=">=$(date -u -v-1d +%F 2>/dev/null || date -u -d yesterday +%F)" \
  --json number,url,repository,author,updatedAt --limit 100
```

The window is since yesterday in UTC, GitHub's clock for `updated:`; it is a cheap pre-filter that cannot miss a request from the last 24 hours, and step 3 does the exact checks. Drop PRs whose author has `is_bot: true`. GitHub removes you from the requested reviewers once you submit a review and adds you back when the author re-requests one, so this list is the PRs waiting on you.

## 3. Skip PRs already handled

For each PR:

```sh
gh pr view <number> -R <owner/repo> --json state,isDraft,headRefOid,reviewRequests,reviews
```

Skip it unless `state` is `OPEN` and `isDraft` is false: the search index lags, so a PR merged or closed minutes ago still shows up as open. Skip it when your login (`gh api user -q .login`, fetched once per pass) is not in `reviewRequests`: the search index lags, and the request may already be gone. Skip it when a review by your login with `state` other than `PENDING` has `commit.oid` equal to `headRefOid`: that covers index lag the other way and a re-request with no new commits, which gets no second review. Skip it when `results/<owner>/<repo>/<number>-<head7>.json` exists: one attempt per head, so a review that failed does not run again in every batch until the author pushes.

Then keep only requests made in the last 24 hours. The search window above is on PR activity, not on the request, so an old request on a PR that got a comment today would slip through it. The PR timeline has the exact event:

```sh
gh api repos/<owner>/<repo>/issues/<number>/timeline --paginate \
  --jq '[.[] | select(.event=="review_requested" and .requested_reviewer.login=="<login>") | .created_at | fromdateiso8601] | max > (now - 86400)'
```

Skip the PR unless that prints `true`.

## 4. Find the clone

For each clone root, check each child directory's `git remote -v`. A match is a remote whose URL ends in `<owner>/<repo>` or `<owner>/<repo>.git`, over https or ssh, on `github.com`. No match: print `skip <owner/repo>#<number>: no local clone` and drop the PR. Never clone.

## 5. Start the batch

When no PR remains, release the lock and print one line that says why: `no new review requests` when step 2 found none, otherwise `<n> requests, none reviewed: <m> handled, <k> without a clone under <roots>`. Otherwise start **one** background Bash command (`run_in_background`) that writes its pid to `lock/pid`, removes `lock` on exit with a `trap`, and reviews the PRs three at a time, oldest `updatedAt` first, one line per PR fed to `xargs`:

```sh
printf '%s\t%s\t%s\n' <clone> <url> <owner>/<repo>/<number>-<head7> ... | xargs -P 3 -L 1 sh -c '
  mkdir -p ~/.cache/pr-review-watch/results/$(dirname "$2")
  cd "$0" && timeout -k 30s 60m claude -p "/tradecraft:pr-review $1 --publish" \
    --permission-mode auto --permission-prompts none \
    --disallowedTools "Edit,Write,NotebookEdit" \
    --strict-mcp-config --max-budget-usd 10 \
    --no-session-persistence --output-format json \
    > ~/.cache/pr-review-watch/results/$2.json 2> ~/.cache/pr-review-watch/results/$2.err'
```

Why each piece: `-P 3` runs three reviews at once, enough to drain a morning's requests without a dozen concurrent agents hitting rate limits, and the next starts when one finishes; `timeout` bounds one stuck review so the lock is not held forever (macOS gets it from coreutils); nobody answers prompts, so `--permission-prompts none` denies instead of hanging; `--disallowedTools` blocks the file-editing tools, the review's own rules cover the shell; `--strict-mcp-config` skips every MCP server, which the review does not use and which cost tokens on every call; `--max-budget-usd` caps one runaway review; `--no-session-persistence` leaves nothing in `/resume`.

One PR failing does not stop the rest. Then print `reviewing <n> PRs: <owner/repo>#<number> ...` and end the turn. Never read a review transcript into this session.

## 6. Report a finished batch

When the background command's completion notice arrives, read only the `Review:`, `Findings:`, and `Highest:` lines from each result file's `.result` (`jq -r .result <file> | grep -E '^(Review|Findings|Highest):'`) and print one line per PR:

```
<owner/repo>#<number>: <Review event>, <findings> findings, highest <severity or none>
```

When the file is not valid JSON or `is_error` is true, print `failed <owner/repo>#<number>` and the paths of the `.json` and `.err` files. When `.result` has none of those lines, `/tradecraft:pr-review` stopped early, for example on a draft; print its first line.
