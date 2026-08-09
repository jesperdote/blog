+++
title = "Recent posts, without the copy-paste"
date = 2026-08-09T13:00:02
description = "Gave the portfolio site a way to show this blog's latest posts on its own, and hit a GitHub PR that reported itself merged while quietly leaving half the change behind."
+++

The portfolio site had a "Recent posts" section listing three of this blog's posts. Hardcoded,
by hand, three `<a>` tags I'd written myself the same afternoon I built the section. Which
meant every new post here was also a manual edit over there — a thing built to sync exactly
never syncing on its own. The fix looked small: give the portfolio a way to ask the blog what's
new, instead of being told once and never again.

## A feed with nothing to say

Zola generates an Atom feed for free — `generate_feeds = true`, done. First build, first look
at the output:

```
<content type="html" xml:base="...">&lt;p&gt;Yesterday I fixed the touchpad going dead
after resume — a &lt;code&gt;systemd-sleep&lt;&#x2F;code&gt; hook that rebinds the driver...
```

Every single entry, full post body, HTML-escaped into a single line. No summary field, no
excerpt, nothing short enough to put in a card on another site. Zola's built-in feed template
only knows about `page.summary` — text extracted from a `<!-- more -->` marker in the post
body — and falls back to the entire `page.content` when that marker doesn't exist. None of the
eleven posts here have one.

## Borrowing the real template instead of guessing

Overriding it meant writing a `templates/atom.xml` that shadows Zola's built-in one — but
guessing at the exact structure risked shipping a subtly-broken feed (wrong element order,
a missing required field, whatever). Pulled the actual template Zola ships, pinned to the exact
version this site runs:

```
gh api "repos/getzola/zola/contents/components/templates/src/builtins/atom.xml?ref=v0.22.1"
```

Then changed exactly one part of it — the summary logic — to check a frontmatter field first:

```jinja2
{% if page.description %}
<summary type="html">{{ page.description }}</summary>
{% elif page.summary %}
<summary type="html">{{ page.summary }}</summary>
{% else %}
<content type="html" xml:base="{{ page.permalink | escape_xml | safe }}">{{ page.content }}</content>
{% endif %}
```

`description` now a required frontmatter field, same tier as `title` and `date`. Backfilled all
eleven existing posts by hand, updated the Telegram bot's own drafting instructions so future
posts don't skip it either — the bot had no idea this field existed until its prompt said so.

## A PR that agreed with an older version of itself

Merged the pull request. Waited a minute for Jenkins to redeploy. Checked the feed:

```
$ curl -I https://.../blog/atom.xml
HTTP/2 404
```

Rebuilt it, redeployed again, same 404. The commit was very clearly on the branch — `git log`
on that branch showed it sitting right there, on top of the one before it. GitHub said the PR
was merged. Neither of those things should have been able to both be true.

```
$ gh pr view 14 --json state,mergedAt,mergeCommit,headRefOid,baseRefOid
{
  "headRefOid": "22f65da...",
  "state": "MERGED"
}
```

`22f65da` was the *first* commit on that branch — the post itself, no feed changes. The second
commit, the one with the actual feature, had a different hash entirely and was pushed to that
same branch a few minutes after the PR's merge had already gone through. GitHub's merge button
carries the head SHA it last saw when the page rendered; my second `git push` landed on the
branch after that page had already been loaded, so the click confirmed a merge against the
version of the branch that existed *before* the push — not after. The API didn't lie. It merged
exactly what it said it merged. That commit simply wasn't the current tip of the branch by the
time the click landed, and nothing forced a re-check in between.

Nothing was lost — the second commit was still sitting right there on the remote branch, fully
intact, just never included in anything. Opened a second PR off the same branch for exactly that
one commit, merged it properly this time, watched the feed actually populate.

## What actually shipped

Post frontmatter gets one more required line:

```
description = "One or two sentences, written like the post itself would say them."
```

That line now does three things at once: feeds this blog's own `/atom.xml`, becomes the summary
a feed reader shows, and is what the portfolio site fetches client-side to render its own three
most recent posts — no hardcoded links, no manual sync, no remembering to go update the other
site. Write the post, write one line of description, push. Two sites update themselves.

The bug in the middle of it wasn't really about feeds or Jinja templates at all. It was a
reminder that "merged" is a claim about a specific commit, not a standing promise to catch up
with whatever you push next — and a UI that shows a stale state with total confidence looks
identical to one that's telling the truth, right up until you go check the thing it was
supposedly telling you about.
