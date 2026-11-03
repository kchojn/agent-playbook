---
description: One pass of the review watcher; finds today's open PRs that request your review and runs /pr-review --publish on each in its own headless session. Run it on a timer with /loop.
argument-hint: <clone root dir>...
---

Run one pass of the review watcher. Start it with:

```
/loop 5m /pr-review-watch ~/code ~/work
```

`$ARGUMENTS` lists the directories that hold your local clones. A clone is a direct child of one of them.

Each review runs in its own `claude -p` process with a fresh context, so this session never holds a review. It sees one summary line per PR. Never use em dashes. Keep each pass short.

State lives in `~/.cache/pr-review-watch/`: `lock` holds the pid of a running batch, and `<owner>-<repo>-<number>.json` holds the last result for a PR.

## 1. Skip while a batch runs

If `lock` exists and `kill -0` on its pid succeeds, print `review batch running` and end the pass. A stale `lock` from a dead pid is removed.

## 2. Find requests

```sh
gh search prs --review-requested=@me --state=open --draft=false --updated=">=$(date +%F)" \
  --json number,url,repository,author,updatedAt --limit 50
```

Drop PRs whose author has `is_bot: true`. GitHub removes you from the requested reviewers once you submit a review and adds you back when the author re-requests one, so this list is the PRs waiting on you.

## 3. Skip PRs already handled

For each PR:

```sh
gh pr view <number> -R <owner/repo> --json headRefOid,reviews
```

Skip it when a review by your login (`gh api user -q .login`, fetched once per pass) has `commit.oid` equal to `headRefOid`. That covers GitHub's search index lag and a re-request with no new commits.

## 4. Find the clone

For each `<root>` in `$ARGUMENTS`, check each child directory's `git remote get-url origin`. A match ends in `<owner>/<repo>` or `<owner>/<repo>.git`, over https or ssh. No match: print `skip <owner/repo>#<number>: no local clone` and drop the PR. Never clone.

## 5. Start the batch

When PRs remain, start **one** background Bash command (`run_in_background`) that creates `~/.cache/pr-review-watch/`, writes its pid to `lock`, removes `lock` on exit with a `trap`, and reviews the PRs one at a time, oldest `updatedAt` first:

```sh
(cd <clone> && claude -p "/pr-review <url> --publish" \
  --permission-mode auto --no-session-persistence --output-format json) \
  > ~/.cache/pr-review-watch/<owner>-<repo>-<number>.json
```

One PR failing does not stop the rest. Then print `reviewing <n> PRs: <owner/repo>#<number> ...` and end the turn. Never read a review transcript into this session.

## 6. Report a finished batch

When the background command's completion notice arrives, read only the `Review:`, `Findings:`, and `Highest:` lines from each result file's `.result` (`jq -r .result <file> | grep -E '^(Review|Findings|Highest):'`) and print one line per PR:

```
<owner/repo>#<number>: <Review event>, <findings> findings, highest <severity or none>
```

When `.result` has none of those lines, `/pr-review` stopped early, for example on a draft; print its first line. When `is_error` is true, print `failed <owner/repo>#<number>` and the file path.

When nothing qualified in steps 2 to 4, print only `no new review requests` and end the turn.
