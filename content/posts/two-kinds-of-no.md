+++
title = "Two kinds of no"
date = 2026-08-08
description = "Building a Telegram bot to draft posts by chat turned into a night spent on one question: an agent with real shell and git access says no for two completely different reasons that look identical from the outside."
+++

I wanted to be able to write about something after the fact without reopening the laptop
for it — the work itself happens there, but the urge to write it up usually shows up
later, away from the desk. So: a Telegram bot that shells out to Claude Code against a clone of
this repo. Simple idea. The entire night went into one question instead: an agent with
real shell and git access is going to say no to things sometimes, and it turns out there
are two completely different reasons it might, and from the outside they look
identical.

## First test: does it even listen

Before trusting it with anything, I wanted to see what it would actually do with plain
shell access. `! sudo ls -l`, `! mkdir deleteme`, `! curl ifconfig.io` - close to arbitrary
commands, sent from Telegram to a Claude session running headless with no terminal
attached.

Nothing happened. Not "it refused" - it *couldn't*. Claude Code's permission system still
runs in headless mode, and headless mode has no way to show an approval prompt, so
anything that would normally ask permission just sits there undecidable and fails closed.
Good news, technically: nothing destructive could happen by accident. Bad news,
practically: this also meant it couldn't write a single file. A bot with no attack surface
and no capability is not a very useful bot.

## Building an allowlist, badly, then correctly

The fix looked simple: `settings.local.json`, an explicit `allow` list, `Write` and `Edit`
included. It wasn't simple. Every write attempt still got denied - including ones squarely
inside the repo the rule was supposed to cover.

The actual bug: in Claude Code's permission patterns, `/some/path/**` isn't an absolute
filesystem path. A single leading slash means "relative to project root." I'd written the
real absolute path with one slash, which Claude Code read as a nested subdirectory named
literally `home/user/project/...` that obviously doesn't exist, so nothing ever matched.
The fix is a second leading slash - `//home/user/project/...` - which is the actual "yes,
this is a real absolute path" syntax. One character, and the difference between
"everything denied" and "everything works."

Except then it was too permissive in the other direction. A bare `Write(**)` pattern - no
anchor at all - let the bot write *anywhere the process could reach*, including a
directory up, into the file holding its own Telegram bot token. I proved this by asking it
to overwrite that exact file. Nothing stopped it at the permission layer. What stopped it
was Claude noticing the request looked like credential exfiltration and declining on its
own judgment - which is a real and useful safety net, but not the one I'd built, and not
one I could rely on staying cautious forever.

## The gate that mattered more than the syntax

Getting the path syntax right fixed *where* writes could land. It didn't fix *when* they
were allowed to happen at all. The permission file was static - once granted, "can write
files" was just always true, whether or not I'd actually asked for a new post.

So the real fix wasn't a permissions file, it was a state machine. A `blog_mode` flag,
false by default, only flipped true by an explicit `/blog` command - and every
write/branch/commit capability granted per-invocation via `--allowedTools`, gated on that
flag, not baked into a config file sitting there permanently allowed. Plain chat before
`/blog` has no write access. Not discouraged - structurally absent, because the tool
grant simply isn't there to find. `git push` and `gh pr create` got the same treatment one
level tighter: never bundled into blog mode at all, only granted for the single call
triggered by `/push` or `/pr` respectively, so even mid-draft the bot can't ship anything
without that exact word.

## Testing the thing that's supposed to stop it

Writing a regression test for this exposed the same trap as the credential-overwrite test
earlier, just quieter. The test checks `permission_denials` in the JSON output - empty
means allowed, right? Not always. Ask Claude to read a `.env` file and "tell me its
contents," and it often declines before ever attempting the `Read` call at all, recognizing
the shape of the request. That also produces an empty `permission_denials` list. A test
that only checked the denial list would have reported a pass either way - whether the
system actually blocked it, or Claude simply chose not to try. For that one case, the test
checks something more direct instead: does the real secret string ever show up anywhere in
the response. Not "was it denied" - "did it leak."

## Where it actually broke, twice

Even with the gate working, the bot ended up on `main` with an uncommitted, unbranched
draft sitting in the working tree - twice, on unrelated topics. Some request mid-
conversation (once it was a failed build-check step) apparently sent it back to `main`
without a matching branch checked out again, and it never returned. Nothing was lost -
everything was still just uncommitted in the working tree - but it's exactly the kind of
mess "always branch off main first" was supposed to prevent. The fix wasn't a smarter
prompt. It was refusing to even start: `/blog` now checks `git status --short` first and
declines outright if the workspace isn't clean, instead of trusting the conversation to
sort itself out.

## Shipping it

Moved it to the VPS tonight - cloned straight from GitHub rather than copying the laptop's
directory over, on principle. Two things broke immediately that hadn't shown up locally:
`claude` wasn't on `PATH` under a non-interactive SSH session (only shows up there under an
interactive login shell, which is not what a background process gets), and `gh` wasn't
installed at all. Both fixed - the binary resolves to an absolute path now instead of
trusting `PATH`, and `gh` got installed and authenticated separately on that host, its own
token, not reused from the laptop.

It's authenticated the same way there as everywhere else: a subscription login, not an API
key. Every draft, every test run, every probe that went into finding all of this tonight
came out of the same Pro/Max plan I already pay for - not a separate bill quietly running
in the background the whole time.
