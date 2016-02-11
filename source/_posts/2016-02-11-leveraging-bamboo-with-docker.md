---
layout: post
title: "Leveraging Bamboo with Docker"
permalink: leveraging-bamboo-with-docker
date: 2016-02-11 21:20:29
comments: true
description: "Leveraging Bamboo with Docker"
keywords: "Bamboo, Docker, Build, Agent"
categories: Docker

tags: Docker, Bamboo, Build

---

My first technical post comes in direct response to a Blog post over on the [Avisi Blog](http://blog.avisi.nl/2016/01/22/get-more-out-of-bamboo-with-docker-part-1 "Bamboo with docker part 1"){:target='_blank'}, where the author correctly notes there is not a lot of available information about effectively integrating Bamboo with Docker.  Through this post I will explain how we formed the Docker base of our Bamboo infrastructure ensuring it is both scalable and resilient to changes over time.

Let me first start off by saying, *we did not start here.*

This has been and continues to be a gradual progression and a long learning process.  The first of many weeks was spent exploring what was even possible with Docker.  A year ago was a very different landscape to what is available today.  We evaluated the possibilities afforded and attempted to ascertain the best fits for our use cases, both in DevOps and developer time.  All along the way Docker has rapidly grown, with more possibilities, documentation, and support throughout the ecosystem.  Our team sits today (and I do mean today, as Docker keeps moving so rapidly) comfortable in what has been created and battle tested through our numerous daily builds.  So with that out of the way, let's get down to the business of the matter.

### Our Trifecta: Docker, Bamboo, CoreOS

Our infrastructure today consists in a 1:1 ratio where a single Bamboo Docker agent resides on a single CoreOS machine.  This is in stark contrast to our previous architecture where we would typically load several Bamboo Agents onto a single VM host.

The question arises then, is this a waste?  Because all of these CoreOS machines only run a single Bamboo Docker agent, the answer is: it depends.  Where before we may have had a build agent (or several) on specific machines to handle specific types of builds due to the dependencies required, Docker frees us from this limitation.  The payoff from using Docker comes when you are able to handle wholly different builds on the exact same box without the headache of worrying what version of a package is installed on the system.  We have spent a great deal of time gathering build requirements (read: trial and error) and either reusing a publically available official Docker container, creating our own from scratch, or something in between.  We can efficiently use every build agent for many jobs and it will not matter what job goes where.

One of of main advantages of moving away from monolithic systems has been to ease the system administration tasks so our team could focus on other areas.  This is one of the main reasons behind our choice of [CoreOS](https://coreos.com){:target="_blank"} for the OS platform.  CoreOS is a stripped down linux distribution utilizing Systemd in which anything beyond a few basic system binaries requires the use of a Docker container.  Our first foray utilizing Docker with builds was done with Ubuntu, and while it solved a problem it still left a headache of ensuring the system (and especially Docker) was up to date.  Not the most elegant of solutions for our purposes as a generic sprawling build infrastructure.  With CoreOS we add user metadata to load our internal CA, start `bamboo-agent-data` and `bamboo-agent` containers, and ensure that the agent itself is up to date.  Any system updates are handled automatically by CoreOS through their use of release channels, greatly easing the burden on the team for day to day administrative tasks.  CoreOS is very fast to boot, very fast to update, and through the metadata config with Systemd unit files makes spinning up new agents a breeze.

So what does this look like in practice?

### High Level

{% responsive_image path: source/assets/images/CoreOS-Bamboo-Agents.png %}

As you may note from the diagram above, our Docker agent is the local orchestrator for new containers outside of its own environment.  A single Bamboo agent and data container agent reside on every CoreOS host.  My first avenue of thinking was attempting the Inception model of Docker within Docker.  This did not work well.  The sibling model has been far easier not only to comprehend and debug, but also from a purely technical standpoint then attempting Inception.  The `bamboo-agent-data` container follows the data container abstraction model, wherein a container resides outside of the main container to persist stateful data.  In the case of Bamboo this is by default the `/root/bamboo-agent-home/xml-data/build-dir`.  Having this data container allows for the sharing of the bamboo working directory between the main bamboo agent and its sibling containers during build time.  Any time a new build is kicked off and containers are created the build scripts typically reference absolute paths `${bamboo.build.working.directory}` in order to obtain the correct data, and return the results to Bamboo.

### Code

Okay so those are high level ideas of how our system is setup to function, what does this look like from the terminal?

So to start things off we have the simple bamboo-data-container:
{% highlight bash %}
docker run -d --name bamboo-agent-data.1 \ 
-v /bamboo-agents/agent.1:/root/bamboo-agent-home/xml-data/build-dir \ 
bamboo-agent-data
{% endhighlight %}

Breaking down this line, we are starting a docker container as a daemon (-d), with a volume mount (-v) from a directory on CoreOS (/bamboo-agents/agent.1) into the volume within the container (/root/bamboo-agent-home/xml-data/build-dir), naming the container for later reference (--name), and finally running the image.

Pretty basic so far, nothing too different than what you might have seen before.

Now onto where the fun begins, the bamboo-agent container:

{% highlight bash %}
docker run -d --name=bamboo-agent \ 
-v /usr/lib/libdevmapper.so.1.02:/usr/lib/libdevmapper.so.1.02 \ 
-v /usr/bin/docker:/usr/bin/docker \ 
-v /var/run/docker.sock:/var/run/docker.sock \ 
--volumes-from=bamboo-agent-data.1 bamboo-agent
{% endhighlight %}

So there are a few interesting items within this line which make the magic of sibling containers function.  Allow me to break this down as before.

We are starting a docker container as a daemon (-d), with several volume mounts (-v), a volumes-from mount (--volumes-from), and finally the name of the image.  

The easy piece to understand is the volumes-from, where we take the volume exposed by our data container and mount it to the same location within the Bamboo agent container.  With that argument we are able to persist build data across reboots, or restarts of the agent container.  This volume also gets mounted to any other sibling container to obtain the checked out source code, or other artifacts from Bamboo's functions.

The truly interesting details are the the rest of the volume arguments.  These provide the basis of allowing our Bamboo agent container not to itself have the Docker binary installed, but instead our dependencies are injected into the agent from the CoreOS machine.  This frees us from caring (for the most part) what version of Docker is running on the CoreOS container.  By injecting everything Docker requires to be run (shared object library, the binary itself, the Docker server socket) every time CoreOS updates to a newer version of Docker and the agent restarts, it automatically takes in those changes without any further administration on our part.

### Wrap up

These three pieces form the basis of our build infrastructure, and leveraging them has been a challenging but fruitful undertaking.  With these building blocks creating isolated builds which report back to Bamboo has been possible, with little administrative overhead as they churn away.



