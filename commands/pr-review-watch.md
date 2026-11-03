---
description: One pass of the review watcher; finds today's open PRs that request your review and runs /pr-review --publish on each. Run it on a timer with /loop.
argument-hint: <clone root dir>...
---

Run one pass of the review watcher. Start it with:

```
/loop 5m /pr-review-watch ~/code ~/work
```

`$ARGUMENTS` lists the directories that hold your local clones. A clone is a direct child of one of them.

Never use em dashes. Keep the output of a pass short: each pass stays in the session's context.

## 1. Find requests

```sh
gh search prs --review-requested=@me --state=open --draft=false --updated=">=$(date +%F)" \
  --json number,url,repository,author,updatedAt --limit 50
```

Drop PRs whose author has `is_bot: true`. GitHub removes you from the requested reviewers once you submit a review and adds you back when the author re-requests one, so this list is exactly the PRs waiting on you.

## 2. Skip PRs already handled

For each PR:

```sh
gh pr view <number> -R <owner/repo> --json headRefOid,reviews
```

Skip it when a review by your login (`gh api user -q .login`, fetched once per pass) has `commit.oid` equal to `headRefOid`. That covers a review that is still in GitHub's index lag and a re-request with no new commits.

## 3. Find the clone

For each `<root>` in `$ARGUMENTS`, check each child directory's `git remote get-url origin`. A match ends in `<owner>/<repo>` or `<owner>/<repo>.git`, over https or ssh. No match: print `skip <owner/repo>#<number>: no local clone` and move on. Never clone.

## 4. Review

Oldest `updatedAt` first, one at a time, call the `pr-review` skill with:

```
<url> --publish --repo <clone path>
```

Do not review two PRs in parallel and do not edit the clone's working tree; `/pr-review` reads commits, not checkouts.

## 5. Report

One line per PR the pass touched:

```
<owner/repo>#<number> <short head sha>: <Review event> <findings> findings, highest <severity or none>
```

`/pr-review` may stop early, for example on a draft; print its one-line reason instead. When nothing qualified, print only `no new review requests` and end the turn.
