---
layout: post
title:  "Docker in Docker"
date:   2019-11-05 15:55:23 -0600
categories: [tooling]
description: "What people actually mean by docker-in-docker."
toc: true
---

# I got beef with all y'alls articles

Oh look, another docker-in-docker post. How original. Ok, I get it, there are
many of these out there. Here's my beef with all the ones I could find at 1am
last night: when people talk about docker-in-docker, they're actually talking
about docker-as-a-service [2] [3] [4] [5] [7] [8] [9] (or, the worse "just
mount docker from your own filesystem" [1] [4] [6] [7]).

That is: docker-in-docker is when you run a container, log into it, and are able
to run `docker`. Docker-as-a-service is when you run a container with `dockerd`
and then linking other containers into that container, which themselves run
`docker`.

_Ok, yeah, I made up those arbitrary distinctions. But, this is my blog, so I_
_can do what I want! :)_

The big difference is that I can do both on my local laptop, but the latter
requires extraorinarily more effort to actually stick in a cloud since most
automated "run a container in the cloud" services have no idea wtf you're
talking about trying to link containers. We're just here to run your
one, single container dude.

# Ok how do you actually do this

Easy solution:

```
$ docker run --privileged -it docker:dind /bin/sh
/ # dockerd-entrypoint.sh &
[...]
/ # docker ps
error during connect: Get http://docker:2375/v1.40/containers/json: dial tcp: lookup docker on 192.168.65.1:53: no such host
/ # # womp womp
/ # unset DOCKER_HOST
/ # docker ps
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
/ # # huzzah!
```

So, yeah, `unset DOCKER_HOST`. What's happening here is that `DOCKER_HOST` is
being set to `tcp://docker:2375`, but we want to unset that to make it use the
unix socket. See more explanation at [https://github.com/docker-library/docker/issues/200](https://github.com/docker-library/docker/issues/200).

# Petty citations

1: https://docs.gocd.org/current/gocd_on_kubernetes/docker_workflows.html#docker-in-docker-dind

2: https://hub.docker.com/_/docker

3: https://docs.gitlab.com/ee/ci/docker/using_docker_build.html

4: https://itnext.io/docker-in-docker-521958d34efd

5: https://sreeninet.wordpress.com/2016/12/23/docker-in-docker-and-play-with-docker/

6: https://jpetazzo.github.io/2015/09/03/do-not-use-docker-in-docker-for-ci/

7: http://blog.teracy.com/2017/09/11/how-to-use-docker-in-docker-dind-and-docker-outside-of-docker-dood-for-local-ci-testing/

8: https://stackoverflow.com/questions/45928958/how-to-run-docker-container-inside-docker-dind

9: https://discourse.drone.io/t/build-and-test-docker-images-with-dind/1808
