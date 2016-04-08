---
layout: post
title: "Building with Docker"
permalink: building-with-docker
date: 2016-04-08 10:13:27
comments: true
description: "Building with Docker"
keywords: "Bamboo, Docker"
categories: Docker

tags: Docker, Bamboo, Build

---

In my previous posts I talk specifically on how and why we use Bamboo, CoreOS, and Docker together to allow our build infrastructure to not only be easily scalable, but also highly agnostic, requiring little dependency management.  In this post I want to expand on exactly how we build and test.

### Recap

Just to quickly recap where we stand in our setup, a CoreOS server has our mounted iso running a cloudinit script.  The script creates and starts our Bamboo agent data container and long running Bamboo Agent container.  When the Bamboo Agent container finishes starting up, it registers finally with Bamboo.  From here our Bamboo agent is ready to accept build requests when our build targets a custom tag isDocker = true.

### Building

Our builds typically utilize scripts within a "ci" directory in each project's repository so different branches can take advantage of the same build and modify the script on each branch should that be neccessary.

Without further ado let's have a look at an example and break it down.

{% highlight bash %}
#!/bin/bash

docker run --rm -v $(docker inspect -f '{{ range .Mounts }}{{ if eq .Destination "/root/bamboo-agent-home/xml-data/build-dir" }}{{ .Source }}{{ end }}{{ end }}' bamboo-agent-data.1)/${bamboo_buildKey}:/home/node node:0.12.13 \
-c "/home/node/node_modules/bower/bin/bower install"

docker run --rm -v $(docker inspect -f '{{ range .Mounts }}{{ if eq .Destination "/root/bamboo-agent-home/xml-data/build-dir" }}{{ .Source }}{{ end }}{{ end }}' bamboo-agent-data.1)/${bamboo_buildKey}:/home/node node:0.12.13 \
-c "/home/node/node_modules/grunt-cli/bin/grunt ci"
{% endhighlight %}

The first thing you might notice is the strange syntax and way of mounting the external data container into the node container.  

Under normal circumstances volume mounting a data container would place the exported directory structure of that data container into the targeted container at the same location (Bamboo home working directory).  e.g. `/root/bamboo-agent-home/xml-data/build-dir` from the data container would map to `/root/bamboo-agent-home/xml-data/build-dir` in the node container.

While this mounting strategy is typically acceptable, we use a customized node container to run under the node user so npm won't yell at us for running commands under root.  Due to this we need to remap the directory to someplace that the node user can access (`/home/node` in this instance).

The key here is we need to gather the actual directory on the server where the data container was originally mounted for the Bamboo Agent.  This could be done statically assuming every Bamboo agent uses the same directory structure across every server, but in case there are several agents on a server or you want to make some modifications it is better to do this programically through the docker inspect cli.

By using docker inspect and appending on the bamboo build key variable, we now have the actual directory on the CoreOS server hosting the current build job.  We can take that and mount it into the node container as you normally would.  From here it is just a matter then of running your specific commands to build and test the job.

In this particular example we are running a bower install proceeded by a grunt command to pull in external bower dependencies and build and test the code.  Both of these tasks are run from within a bash script, created as sibling containers from within the Bamboo Agent container.

### Wrap up

Taking the lessons learned from the previous posts and applying them to actually do some work is trivial now that we have our solid base of CoreOS and Docker setup and running.  Although I used a node container as an example this could be any container which runs as a binary, a good template to move forward.  The biggest takeaway from this post and example is to remember the Bamboo Agent container connects to all sibling containers via the data volume container, so remembering to mount that data container is a very important step in the process.