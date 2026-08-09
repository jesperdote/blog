+++
title = "The deploy that kept saying done"
date = 2026-08-07T20:48:34
description = "Moving this blog off Netlify to a self-hosted Jenkins pipeline on a BananaPi took four separate bugs, each one hiding behind the last one's fix."
+++

This site used to deploy to Netlify. I wanted it self-hosted instead, behind the same
Jenkins that already deploys the homelab's observability stack, running on a BananaPi
that already does six other things. On paper: clone the repo, build it, serve it,
proxy it. A couple hours, tops.

It took four separate bugs, each hiding behind the last one's fix, before a single push
actually landed on the site.

## Two agents, immediately incompatible

Jenkins already had one SSH agent — the VPS itself. Adding the BananaPi as a second one
looked identical: create a user, drop in an SSH key, point a new node at it. First
connection attempt:

```
Error: LinkageError occurred while loading main class hudson.remoting.Launcher
    java.lang.UnsupportedClassVersionError: hudson/remoting/Launcher has been compiled
    by a more recent version of the Java Runtime (class file version 61.0), this
    version of the Java Runtime only recognizes class file versions up to 55.0
```

Class file 55 is Java 11, which is what the BananaPi had. Class file 61 is Java 17.
Installed 17, tried again — further, but not there:

```
Caused by: java.lang.UnsupportedClassVersionError: hudson/slaves/SlaveComputer$SlaveVersion
    has been compiled by a more recent version of the Java Runtime (class file version 65.0),
    this version of the Java Runtime only recognizes class file versions up to 61.0
```

65 is Java 21. Turns out the agent doesn't just need to satisfy `remoting.jar`'s own
minimum — the controller ships some of its own runtime classes to the agent live, over
the wire, compiled at whatever JDK built the controller's container image. That image is
`jenkins/jenkins:latest-jdk21`. Installed 21, pointed the node's Java Path straight at
that binary instead of the system default, and the channel finally opened clean.

## A job that existed and didn't

Added a new pipeline job by editing the Jenkins-Configuration-as-Code file and restarting
the container. The boot log showed it get created:

```
INFO j.j.plugin.JenkinsJobManagement#createOrUpdateConfig: createOrUpdateConfig for deploy-blog
```

The job's `config.xml` was sitting on disk, fully valid. It was not in the UI. Direct URL
to it: 404. Jenkins' own Script Console settled it for good —

```groovy
println(Jenkins.get().getItem('deploy-blog'))
// null
println(Jenkins.get().items.collect { it.name })
// [apply-grafana-alerts, apply-thanos-rules, deploy-server]
```

— the job genuinely didn't exist as far as the running process was concerned, despite
its own log saying otherwise. The create call landed something like 66 milliseconds
before Jenkins' "load all jobs from disk" boot phase — close enough that a plain
`docker compose restart` reproduced the missing job every time. `docker compose up -d
--force-recreate` didn't.

## An architecture the registry forgot

With the agent and the job both finally real, the first build got as far as actually
building:

```
+ docker run --rm -u 1005:1005 -v /home/jenkins/repos/blog:/app --workdir /app \
    ghcr.io/getzola/zola:v0.22.1 build
Unable to find image 'ghcr.io/getzola/zola:v0.22.1' locally
docker: no matching manifest for linux/arm/v7 in the manifest list entries.
```

The BananaPi is 32-bit ARM. Zola's official image only ships `amd64` and `arm64` —
confirmed with `docker manifest inspect`, and their GitHub releases don't publish an
armv7 binary either, so there was no native fallback. Compiling a Rust static-site
generator from source on a 2GB-RAM single-board computer wasn't an appealing plan B.

The actual fix: split the pipeline. Build runs on the VPS agent instead, which is a
normal `x86_64` box the image already supports, then `stash`es the built `public/`
directory across to the BananaPi agent for the deploy step. Two machines, two
architectures, one pipeline.

## A directory the container forgot

Everything green. Site live. Then I published this post's predecessor, the pipeline ran
again, went green again — and the site started 403ing, on the direct port and through
the proxy both. nginx's own log explained the shape of it:

```
directory index of "/usr/share/nginx/html/" is forbidden
```

Forbidden, not missing — nginx couldn't find an index file to serve, even though the
host directory clearly had one:

```
$ docker exec blog ls -la /usr/share/nginx/html/
total 0
```

Completely empty, from the container's point of view. The deploy step does `rm -rf
public` before unpacking the freshly built site — clean slate, in theory. In practice
that deletes the directory outright and lets the unpack step recreate it from scratch,
which means a *new* directory at the same path, with a new inode. The container's bind
mount had already been established against the old one at startup, and a bind mount
doesn't follow a path — it follows the specific thing it was pointed at. Delete that out
from under a running container and it just keeps looking at where its inode used to be,
which is now nothing.

Same shape of bug as the missing job, one layer down the stack: state that looks live
but was actually established before the change happened, and nothing forces it to
notice. `docker-compose up -d --force-recreate` again, same reasoning — recreate the
container so its bind mount gets re-established against whatever's actually there now.

## Is it simple?

The shape of it, yes: poll GitHub, build, stash, deploy, four stages. No test gate, no
staging environment, no rollback, a single `echo` for a failure notification. That part
took about twenty minutes to write.

None of the four bugs were in that shape. They were all in the gap between "this should
just work" and the specific, unglamorous fact of the machines involved — one running a
JDK five major versions behind the controller, one running Docker Compose from 2020, one
that doesn't get to run `amd64` images, and a kernel-level assumption about what a bind
mount actually promises to track. The pipeline is simple. Getting it to hold still on
real hardware wasn't.
