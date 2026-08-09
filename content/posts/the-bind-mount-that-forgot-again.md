+++
title = "The bind mount that forgot again"
date = 2026-08-09
description = "Extracting a second site into its own Jenkins pipeline hit two bugs already documented from the first one, plus a new one that had nothing to do with infrastructure at all."
+++

Giving the portfolio site the same treatment as this blog — its own repo, its own Jenkins
pipeline, deployed to the BananaPi the same way — should have been the fast path. The
blog's own migration is one post back in this archive: four bugs, fully diagnosed,
runbook written. Extract a repo, wire a pipeline, done in twenty minutes.

It hit two of those same four bugs again, plus a new one that had nothing to do with
infrastructure at all.

## Two branches, two different truths

The portfolio lived as a subfolder in a monorepo that also holds the reverse-proxy
config, alongside a handful of other self-hosted services. Before touching anything,
`git log` turned up two branches that had quietly diverged: `main`, and a feature branch
still mid-flight for some n8n routing work.

Both had touched the exact same two things, in incompatible ways. The feature branch
disabled the portfolio's route entirely and blanket-routed all unmatched traffic to n8n.
`main` had, independently, grown a properly scoped set of n8n paths, kept the portfolio's
route alive, and defaulted unmatched traffic to a plain 404 — objectively the better
config, and the one that turned out to actually be live.

Worse: `main` also had its own independent rewrite of the portfolio's homepage, a
completely different persona than the one from that session's work. Same file, two
unrelated rewrites, no merge between them. There was no clean way to reconcile both —
one had to lose. Kept the version freshly built against an actual resume; the other,
untethered from anything real, got dropped.

Lesson that should've been obvious going in: check the deployed branch's *actual* state
before assuming the branch you're sitting on is the one anything is building on top of.

## Splitting a repo without losing its history

The extraction itself was the easy part, `git subtree split` handles it without any extra
tooling:

```
git subtree split --prefix=whoami -b whoami-extract
git clone -b whoami-extract --single-branch <monorepo> <new-repo-dir>
```

Full commit history for just that subfolder, rewritten so it becomes the new repo's root.
Pushed it up, added a `Jenkinsfile` copied from the blog's pattern — actually simpler this
time, since the portfolio's Dockerfile is a plain `nginx:alpine` build with nothing
architecture-specific in it. No need to split build and deploy across two agents the way
the blog has to for its ARM-incompatible static site generator; one stage, on the BananaPi
agent, does the whole thing.

Then: remove the now-redundant subfolder from the monorepo.

## The bind mount, again

The live site went straight to 403 the moment that folder was gone.

Turned out the container that had been serving the portfolio in production this whole
time wasn't running from a built image at all — it was started from the *dev*
compose file, the one meant for live-editing on a laptop, bind-mounting the source
directory straight into nginx's html root. Deleting that directory out from under a
running bind mount doesn't just remove old files; it removes the thing nginx is looking
at, full stop. Directory listing forbidden, because there's nothing left to list.

Not a new failure mode — just a new place for an old one to hide. The dev compose file
had apparently been what got run by hand, once, to get the site live in the first place,
and nobody ever came back to swap it for the real production build. Fixed properly this
time: cloned the new repo to the path Jenkins actually deploys from, built the real image,
brought that container up instead.

## Then the front-proxy did it too

With the portfolio itself sorted, the last piece was pointing the reverse proxy at the new
route name. Edited the config, `git pull`ed it onto the box, ran `nginx -s reload` — and
the live route kept 404ing anyway.

```
$ md5sum /path/on/host/default.conf
e30ea0132ae9e95347fd612760c3bb2f
$ docker exec nginx md5sum /etc/nginx/conf.d/default.conf
461cb6b4b438ed5e699b04436de4c9bd
```

Two different files, same bind-mounted path. This is the exact bug from the blog's own
deploy post, just wearing a different container's name: `git pull` doesn't edit a file in
place, it replaces it, and a fresh replacement gets a fresh inode. A bind mount doesn't
track the path — it tracks the specific inode it was pointed at when the container
started, twelve days earlier in this case. `reload` re-reads nginx's config from whatever
that stale view still points to, which was never going to be the new file no matter how
many times it ran.

Already had the fix written down from last time. Recreate the container, not just
reload it:

```
docker-compose up -d --force-recreate
```

md5s matched immediately after.

## The job that (almost) didn't exist, again

Last one: registering the new pipeline in Jenkins by editing the Configuration-as-Code
file and recreating the controller. The boot log showed the exact success line —

```
INFO j.j.plugin.JenkinsJobManagement#createOrUpdateConfig: createOrUpdateConfig for deploy-whoami
```

— `config.xml` sitting valid on disk. Not in the UI. The previous post on this exact
failure mode says, in its own words, that `--force-recreate` "reliably avoids this."
It didn't, this time — needed a second recreate before the job actually showed up as a
live item instead of just a file that Jenkins had technically written and then not
loaded. "Reliably" was doing more work in that sentence than it should have.

## Same house, same bugs

None of this was new territory. Two of these three failures were bugs already caught,
diagnosed, and written up in detail, on this exact stack, less than two weeks ago — and
they still happened again, because the fix was never "don't hit this bug," it was
"here's what to do when you hit it." A bind mount doesn't get less literal about tracking
inodes because you've met it before. A JCasC seed race doesn't get less racy because you
know its name. Infrastructure quirks aren't puzzles that stay solved once you've solved
them once; they're properties of the specific machines involved, and they come back every
single time the pattern that triggers them repeats — runbook in hand or not.
