---
layout: post
title: "Bamboo Agent Enhancements"
permalink: updating-bamboo-mounts
date: 2016-12-15 12:50:02
comments: true
description: "Updating Bamboo Mounts"
keywords: "Bamboo, Docker""
categories: Docker

tags: Docker, Bamboo, Build

---

After several months of runtime with ~30 Bamboo agents running within their respective containers on the CoreOS hosts and many builds to their name, the setup I had configured for use was running into several issues.  Namely, the ability to track failures due to host or agent issues and apply Bamboo server side configurations (and have them persisted).  Tackling these was not too difficult but was of course a learning experience digging into the Docker documentation.

### Hostname

The setup I had posted about previously would automatically spin up a Bamboo agent each time a CoreOS server would start, but would be registered into Bamboo as the container's hostname.  This became an irritant because we could not identify which CoreOS host an agent was running.  The solution was fairly simple, passing in the HOST environment variable  ( `-e HOST=%H` ) was enough for the Bamboo Agent to utilize the external host Host's name.  In addition to the environment variable, the container hostname can also be set with the `-h` flag at runtime.  We currently employ both methods to ensure it is available to both the container and the Bamboo agent.

While this was an easy solution a major problem still remained, holding us back from utilizing Bamboo's built in capabilities to assign custom parameters to an individual agent.  Everytime an Agent comes back online it registers as a new agent, even when the host name is passed into the container startup.  This led to the second fix.

### Persistence

Originally all that was required for our builds to function correctly with sibling containers was to persist only the `/root/bamboo-agent-home/xml-data/build-dir` in order to mount that directory into the other container(s).  This was a great way of accomplishing having the Bamboo agent install at Docker image build time, maintain our custom capabilities, and have the external mount point for sibling containers.  What this did not account for however was requiring the Bamboo Agent's id to persist across container restarts.  This required a larger change in not only how the container is built, but also how the container is started.

Previously our Dockerfile (and image consequently) installed the Bamboo agent and required no startup script.  This has been changed to add the Bamboo agent jar file to the root of the image to be installed and started by the new startup script.  Additionally, we now use [Yelp's dumb-init](https://github.com/Yelp/dumb-init "Yelp Dumb-init"){:target='_blank'} to kick off the startup script.  Finally the exported volume mount has been moved up several directory levels to maintain persistent agent data.

{% highlight bash %}
FROM ubuntu:14.04

ENV HOME /root

ENV DEBIAN_FRONTEND noninteractive

RUN echo "America/Chicago" | sudo tee /etc/timezone && dpkg-reconfigure tzdata && \
    apt-get update && apt-get install -y software-properties-common && \
    add-apt-repository ppa:openjdk-r/ppa && \
    apt-get -qq update && \
    apt-get install --ignore-missing --no-install-recommends -y openjdk-8-jdk git unzip zip wget openssh-client openssl && \
    apt-get clean && rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/* && \
    wget --no-check-certificate -O /usr/local/bin/dumb-init https://github.com/Yelp/dumb-init/releases/download/v1.2.0/dumb-init_1.2.0_amd64 && \
    chmod +x /usr/local/bin/dumb-init

COPY atlassian-bamboo-agent-installer-5.14.1.jar /atlassian-bamboo-agent-installer-5.14.1.jar
COPY startup.sh /
COPY wrapper.conf /
COPY bamboo-capabilities.properties /

VOLUME ['/var/run/docker.sock','/root/bamboo-agent-home', '/bin/docker']

ENTRYPOINT ["/usr/local/bin/dumb-init"]
CMD ["/startup.sh"]

{% endhighlight %}

And the startup script now consists of:

{% highlight bash %}

#!/bin/bash -xe

java -jar /atlassian-bamboo-agent-installer-5.14.1.jar https://bamboo.jamf.build/agentServer install
cp /wrapper.conf /root/bamboo-agent-home/conf/wrapper.conf
cp /bamboo-capabilities.properties /root/bamboo-agent-home/bin/bamboo-capabilities.properties

/root/bamboo-agent-home/bin/bamboo-agent.sh console

{% endhighlight %}

The actual command line startup of the agent has not been changed; however, since we create and mount a data container and we have moved the volume mount up several directory levels, the Bamboo agent data is persisted within the data container, and the actual build directory is still externally mounted to the local file system for other sibling containers to utlize.

### Wrap Up

With a few small tweaks to how the Bamboo agent image is created and started we now have the ability (as we did before with static agents), to define custom parameters, dedicate agents, have user friendly names, and better track potential build server issues.