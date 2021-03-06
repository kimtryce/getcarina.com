---
title: "Moving ChatOps Into Carina"
date: 2016-03-15 16:00
comments: true
author: Nick Silkey <nick.silkey@rackspace.com>
published: true
excerpt: >
  The Rackspace Cloud DNS team migrated their ChatOps workloads into Carina with great success and little effort.
categories:
 - Deployment
 - Carina
 - Docker
 - Production
 - ChatOps
 - Heroku
 - ObjectRocket
authorIsRacker: true
---

I am a systems engineer on the Rackspace Cloud DNS product team.  We are a small nimble team who have been able to move [quickly and accurately](https://speakerdeck.com/filler/the-ops-must-be-crazy-hack-your-teams-ops-culture-with-one-weird).  Like many other teams within Rackspace, we have been leveraging [ChatOps](https://speakerdeck.com/jnewland/chatops-at-github) as a tool to both [enable clear communications](https://www.pagerduty.com/blog/what-is-chatops/) and [automate routine tasks](http://blog.flowdock.com/2014/11/11/chatops-devops-with-hubot/) in the open for full transparency across the team.

## The Preamble

Our ChatOps journey began like most teams in that we quickly spun up a proof-of-concept [Hubot](https://hubot.github.com/) instance on [Heroku](https://www.heroku.com/home).  The barrier to entry was really low and the deployment story was great.  We were able to track our ChatOps bot instance as version-controlled code, with peer review, with options to roll updates in a streamlined fashion to maximize availability.  We were quickly enamored with not only that but all the other values that ChatOps brings to the table.

As our [product development matured](http://blog.rackspace.com/sign-now-early-access-rackspace-managed-dns-powered-openstack/), we realized that we needed to better [own the availability](http://www.whoownsmyavailability.com/) of our Hubot instance .  We were running on a skunkworks, no-cost plan at Heroku.  Moreover Heroku recently changed its service offering to enforce that said free plans enforce a ['sleep' policy](https://devcenter.heroku.com/articles/dyno-sleeping) on apps hosted in their fleet.  This means that there would be times where our instance would be unavailable.  Not good for a 3am on-call issue where ChatOps is integral to your team's incident response workflows!  While there are [some tactics to help maximize availability of your instance](https://github.com/hubot-scripts/hubot-heroku-keepalive), we realized that we would have to change things up for something so integral to our team's operations.

Our initial thought was to bring the Hubot instance and its redis-based datastore into a Rackspace Public Cloud instance.  We started assembling the necessary configuration management code to do the needful.  However at the OpenStack Summit in Tokyo we got to see how easy Carina was to both get started with and to manage the lifecycle of container-based applications.

![A Bright Idea]({% asset_path 2016-03-15-moving-chatops-into-carina/lloyd-idea.gif %})

## Why Not Carina?

Our team rallied behind the idea of moving our ChatOps instance into Carina.  We were reasonably savvy using [Docker](https://www.docker.com/) for things like peer-reviewing the team's [Jenkins Job Builder](http://docs.openstack.org/infra/jenkins-job-builder/) changes.  Also [one of our developers](http://ionrock.org/) was able to bring a [proof of concept of our stack](https://github.com/rackerlabs/designate-carina), powered by [OpenStack Designate](http://docs.openstack.org/developer/designate/), into Carina via [`docker-compose`](https://docs.docker.com/compose/).

One thing we were still having to evaluate was what to do about the persistent datastore component of Hubot's redis-brain.  We were using the free RedisToGo offering bundled with Heroku.  One option was using a [data container running redis in Carina](https://getcarina.com/docs/tutorials/data-stores-redis/) linked to a ChatOps container via docker-swarm networking.  The other option on the table was leveraging [ObjectRocket's hosted redis service](http://objectrocket.com/).  Since ObjectRocket is considered best of breed when it comes to datastore availability and management, we opted for that solution.

One afternoon a team member bit off the story in the product's backlog to migrate our ChatOps instance from Heroku to Carina.  The nature of the task was to shutdown the Heroku-hosted instance, backup the RedisToGo-hosted datastore, restore the datastore dump into ObjectRocket, build a ChatOps Docker container, and run said container in Carina.

And it literally took under an hour to do all that.  Color me impressed.

![Color me impressed]({% asset_path 2016-03-15-moving-chatops-into-carina/aha.gif %})

## Migrating To Carina

The RedisToGo instance's connection information was available as the `REDISTOGO_URL` environment variable of the Hubot instance via the [`heroku-toolbelt`](https://toolbelt.heroku.com/) by way of a `heroku config --app=${APP}`.  Using this connection string information, we were able to use the `redis-cli` to pull down an `rdb` dump of the redis instance which powered Hubot's redis-brain.

```bash
# Set some env vars
DUMP=designate.hubot.`date +%Y%m%d`.rdb
HOST=host.redistogo.com
PORT=10411
PASS=awesomerandopass

# Dump redis to a local backup
redis-cli \
-h ${HOST} \
-p ${PORT} \
-a ${PASS} \
--rdb ${DUMP}
```

After successfully capturing the dump, we verified the state of the dump was healthy via `redis-check-dump`:

```bash
redis-check-dump ${DUMP}
```

Finally, we provisioned a redis instance in ObjectRocket.  At the time of this blog, ObjectRocket's redis offering allows for a two-node [redis sentinel](http://redis.io/topics/sentinel) instance configured for high-availability.  It is available in the three US-based Rackspace regions and the London region.

One thing that contrasts ObjectRocket's redis offering from RedisToGo is that its ingress filtered by default ... which is a good thing!  Once provisioned we were able to use the ObjectRocket panel to allow an ingress permit from the host where we captured the RedisToGo dump to.  Using [`rdbtools`](https://github.com/sripathikrishnan/redis-rdb-tools), we were able to push our rdb-based dump into our newly-provisioned ObjectRocket redis instance:

```bash
HOST=lengthyhash.publb.rackspaceclouddb.com
PORT=6379
PASS=awesomerandopass

rdb --command protocol ${DUMP} | \
redis-cli \
-h ${HOST} \
-p ${PORT} \
-a ${PASS} \
--pipe
```

With the redis-brain migrated, we set out to build the hubot container for use in Carina.

## We Can Rebuild Him!

As was done with the `heroku config` function to retrieve the `REDISTOGO_URL` environment variable from the Heroku-based Hubot instance, we were able to retrieve all the other bootstrapped environment variables of varying use.  Examples here include GitHub, Jenkins, PagerDuty, and Slack related strings necessary for their respective integrations.

With all those configuration values, we were able to setup a simple `Dockerfile` which setup all that was necessary to run the Hubot instance in Carina.

```
FROM node:4
MAINTAINER Cloud DNS, dnsaas@rackspace.com

ENV HUBOT_GITHUB_ORG                    rackerlabs
ENV HUBOT_GITHUB_TOKEN                  somegithubtoken
ENV HUBOT_RACKSPACE_API                 someracktoken
ENV HUBOT_PAGERDUTY_USER_ID             somepagertoken
ENV HUBOT_SLACK_TOKEN                   someslacktoken
ENV REDIS_URL                           redis://someredispass@someobjectrockethash.publb.rackspaceclouddb.com:6379/

ENV BOTDIR /opt/bot

RUN mkdir ${BOTDIR}

COPY package.json ${BOTDIR}
WORKDIR ${BOTDIR}
RUN npm install
COPY . ${BOTDIR}

CMD bin/hubot -a slack
```

Voila!

One other thing to note is that you do have to set another ingress permit from your swarm cluster in Carina into your ObjectRocket redis instance.  As mentioned, its firewalled off by default.  You can grab the IPv4 egress IP of your Carina cluster via a `docker info`.  Taking that IPv4 address to the ObjectRocket control panel to permit access into your hosted Redis instance should be all you need to have your Hubot instance talking to ObjectRocket-hosted redis-brain!

## `make` All The Things!

One thing my team has some love for is using `Makefile`s in order to automate commonly-used tasks.  We setup some simple `make` targets which aim to help streamline Carina-based operations as they relate to Hubot.

It assumes you've bootstrapped your shell's runtime using the necessary `carina credentials` and `carina env` commands:

```bash
SHELL := /bin/bash

CONTAINER = designate-hubot

help:
	@echo ""
	@echo "setup       - setup carina client via homebrew"
	@echo ""
	@echo "create      - create carina cluster"
	@echo "delete      - delete carina cluster"
	@echo ""
	@echo "info        - show carina stats via docker info"
	@echo "ps          - check on running docker containers"
	@echo ""
	@echo "build       - build $(CONTAINER) docker image"
	@echo "run         - run $(CONTAINER) docker container"
	@echo "roll        - kicks off stop, build, start"
	@echo ""
	@echo "start       - start $(CONTAINER) docker container"
	@echo "stop        - stop $(CONTAINER) docker container"
	@echo "kill        - stop $(CONTAINER) docker container"
	@echo "rm          - remove $(CONTAINER) docker container"
	@echo "clean       - remove all exited docker containers"
	@echo "restart     - kicks off kill, start"
	@echo ""
	@echo "logs        - pulls logs from $(CONTAINER) docker container"
	@echo ""

check-vars:
ifndef CARINA_USERNAME
	$(error CARINA_USERNAME is undefined!)
endif
ifndef CARINA_APIKEY
	$(error CARINA_APIKEY is undefined!)
endif

setup:
	brew install carina

create: check-vars
	@carina create --segments=1 --autoscale --wait $(CONTAINER)

delete: check-vars
	@carina delete $(CONTAINER)

info: check-vars
	@docker info

ps: check-vars
	@docker ps -a

build: check-vars
	@docker build -t $(CONTAINER) .

run: check-vars
	@docker run --name $(CONTAINER) -d $(CONTAINER)

rm: check-vars
	@docker rm $(CONTAINER)

clean: check-vars
	@$(shell docker ps -a | grep Exited | awk {'print $1'} | xargs -n1 -t docker rm)

start: check-vars
	@docker start $(CONTAINER)

stop: check-vars
	@docker stop $(CONTAINER)

kill: check-vars
	@docker kill $(CONTAINER)

roll: stop build start

restart: stop start

logs: check-vars
	@docker logs $(CONTAINER)

compose:
	@docker-compose up -d
```

This allows us to do some powerful things via some simple `make` targets which are close to the code.

```bash
$ make

setup       - setup carina client via homebrew

create      - create carina cluster
delete      - delete carina cluster

info        - show carina stats via docker info
ps          - check on running docker containers

build       - build designate-hubot docker image
run         - run designate-hubot docker container
roll        - kicks off stop, build, start

start       - start designate-hubot docker container
stop        - stop designate-hubot docker container
kill        - stop designate-hubot docker container
rm          - remove designate-hubot docker container
clean       - remove all exited docker containers
restart     - kicks off kill, start

logs        - pulls logs from designate-hubot docker container

$ make build
Sending build context to Docker daemon  85.5 kB
Step 1 : FROM node:4
 ---> 3538b8c69182
...
Step 55 : CMD bin/hubot -a slack
 ---> Using cache
 ---> 13115b0a1a31
Successfully built 13115b0a1a31
$ docker ps
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS               NAMES
2c7497c66de7        carina/consul       "/bin/consul agent -b"   7 days ago          Up 7 days                               5b26b03f-f3ed-4b0c-835e-2d69745aacb4-n1/carina-svcd
$ make run
a476ac493bd27a744f34714845acc42c7c644652b780e012a1c6e010d050fa77
$ make ps
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS               NAMES
f444c6888322        designate-hubot     "/bin/sh -c 'bin/hubo"   6 seconds ago       Up 1 seconds                            5b26b03f-f3ed-4b0c-835e-2d69745aacb4-n1/designate-hubot
82498aee00d1        swarm:1.1.0         "/swarm manage -H=tcp"   7 days ago          Up 7 days                               5b26b03f-f3ed-4b0c-835e-2d69745aacb4-n1/swarm-manager
84f8c076d7ff        swarm:1.1.0         "/swarm join --addr=1"   7 days ago          Up 7 days                               5b26b03f-f3ed-4b0c-835e-2d69745aacb4-n1/swarm-agent
78e398317263        cirros              "/sbin/init"             7 days ago                                                  5b26b03f-f3ed-4b0c-835e-2d69745aacb4-n1/swarm-data
2c7497c66de7        carina/consul       "/bin/consul agent -b"   7 days ago          Up 7 days                               5b26b03f-f3ed-4b0c-835e-2d69745aacb4-n1/carina-svcd
073068bcc147        cirros              "/sbin/init"             7 days ago                                                  5b26b03f-f3ed-4b0c-835e-2d69745aacb4-n1/carina-svcd-data
$ make stop
designate-hubot
$ make ps
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS                       PORTS               NAMES
f444c6888322        designate-hubot     "/bin/sh -c 'bin/hubo"   40 seconds ago      Exited (137) 1 seconds ago                       5b26b03f-f3ed-4b0c-835e-2d69745aacb4-n1/designate-hubot
82498aee00d1        swarm:1.1.0         "/swarm manage -H=tcp"   7 days ago          Up 7 days                                        5b26b03f-f3ed-4b0c-835e-2d69745aacb4-n1/swarm-manager
84f8c076d7ff        swarm:1.1.0         "/swarm join --addr=1"   7 days ago          Up 7 days                                        5b26b03f-f3ed-4b0c-835e-2d69745aacb4-n1/swarm-agent
78e398317263        cirros              "/sbin/init"             7 days ago                                                           5b26b03f-f3ed-4b0c-835e-2d69745aacb4-n1/swarm-data
2c7497c66de7        carina/consul       "/bin/consul agent -b"   7 days ago          Up 7 days                                        5b26b03f-f3ed-4b0c-835e-2d69745aacb4-n1/carina-svcd
073068bcc147        cirros              "/sbin/init"             7 days ago                                                           5b26b03f-f3ed-4b0c-835e-2d69745aacb4-n1/carina-svcd-data
$ make rm
designate-hubot
$ make ps
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS               NAMES
82498aee00d1        swarm:1.1.0         "/swarm manage -H=tcp"   7 days ago          Up 7 days                               5b26b03f-f3ed-4b0c-835e-2d69745aacb4-n1/swarm-manager
84f8c076d7ff        swarm:1.1.0         "/swarm join --addr=1"   7 days ago          Up 7 days                               5b26b03f-f3ed-4b0c-835e-2d69745aacb4-n1/swarm-agent
78e398317263        cirros              "/sbin/init"             7 days ago                                                  5b26b03f-f3ed-4b0c-835e-2d69745aacb4-n1/swarm-data
2c7497c66de7        carina/consul       "/bin/consul agent -b"   7 days ago          Up 7 days                               5b26b03f-f3ed-4b0c-835e-2d69745aacb4-n1/carina-svcd
073068bcc147        cirros              "/sbin/init"             7 days ago                                                  5b26b03f-f3ed-4b0c-835e-2d69745aacb4-n1/carina-svcd-data
$ make run
b4f9d7017392dc385db3238ac4a8cf356ff59f8d3721df3cb0d1d89ca11d5c61
$ make roll
designate-hubot
Sending build context to Docker daemon  85.5 kB
Step 1 : FROM node:4
 ---> 3538b8c69182
...
Step 55 : CMD bin/hubot -a slack
 ---> Using cache
 ---> 6c75c94c5d97
Successfully built 6c75c94c5d97
designate-hubot
```

This makes it trivial to hand these operations over to continuous integration in order to allow code changes to Hubot to lifecycle manage our always running ChatOps container in Carina!

One final thing we do is set a `.dockerignore` in the Hubot repository to keep cruft which doesnt need to go to the Docker daemon from going.  This keeps the 'context' thats sent to the Docker daemon lean for more rapid builds, rolls, etc.

Our example `.dockerignore`:

```
.git
node_modules
```

## Onward and Upward

Some additional stuff we would like to do is fold the shell-prep steps into the Makefile as appropriate.  And also perhaps move the ENV setting as is done in the `Dockerfile` to something like service discovery with Consul.

Another thing we are interested in pursuing is the idea of moving configuration management operations into ChatOps.  Our team uses a combination of [Chef](https://www.chef.io/) and [Ansible](https://www.ansible.com/) to achieve both persistent declarative state and imperative orchestration operations respectively.  Carina enables our team to bring tooling like the [chefdk](https://downloads.chef.io/chef-dk/) and the relevant [`knife.rb`](https://docs.chef.io/config_rb_knife.html) and related keys into the container and have [chat-based `knife`](https://github.com/hubot-scripts/hubot-chef) operations succeed.  Quick proof of concept attempts at making this work are promising (eg "@hubot knife role show rax_resolver")!

![Impressive!]({% asset_path 2016-03-15-moving-chatops-into-carina/impressive.gif %})

Overall the Cloud DNS product team was extremely impressed with how easy it was to get started using Carina.  Also the deployment story and workflows that teams can use to manage their containers in their swarm clusters is pretty powerful stuff.  The turnkey nature of Heroku is almost completely captured by Carina in addition to allowing for some deeper functionality if desired!
