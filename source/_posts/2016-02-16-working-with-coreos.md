---
layout: post
title: "Working with CoreOS"
permalink: working-with-coreos
date: 2016-02-16 10:10:55
comments: true
description: "Working with CoreOS"
keywords: "CoreOS, Bamboo"
categories: CoreOS

tags: CoreOS, Bamboo, Build

---

In my previous post I mentioned [CoreOS](https://coreos.com "CoreOS"){:target="_blank"} as being one of our key components within our Dockerized build infrastructure.  I wanted to spend some time to elaborate on how we utilize CoreOS to ease our system administrative overhead and allow for (near) effortless scaling both inside and outside the firewall.

Our initial foray into utilizing Docker for builds was to install it on one of our current Ubuntu build servers.  Having just the Docker binary installed and the Bamboo agent running normally functions exactly as you would expect.  From your build scripts you can call out to the Docker binary and spin up some container(s), run it, and collect your results from the bamboo build working directory.  

While this did function, the issue of system administration was left unresolved and maintaining the version of Docker installed, not only against this one particular server but all of the other servers as well was not something I wanted to spend my time doing.  These types of administrative tasks are exactly what I want to use Docker to get away from doing on a day to day basis.  

I will note that a number of these time consuming system administrative tasks can be mitigated utilizing Ansible to run playbooks and maintain consistency across the servers, but if all I want to do is effectively manage the Docker version installed this seemed overkill.  My goal in all of this was not to merely add Docker as another option, I wanted it to be the only option for running builds.  

There must be a better option than a full blown Ubuntu for this task.

### Enter CoreOS

CoreOS can be summed up partly as a stripped down Linux distribution, supporting Docker out of the box and really nothing else beyond that.  This fact alone intrigued me as all I was really after was something to run Docker on and not have to concern myself with other potentially out of date system packages.

CoreOS System updates are handled based upon the release channel chosen (Stable, Beta, or Alpha).  These system updates are by default automatically applied and restart the system once the update has completed.  Due to the size of the OS and how updates are performed, these operations are very fast to complete and roll back safe should something go awry.  You can read more about how the update system works here: [CoreOS Updates](https://coreos.com/using-coreos/updates/ 'Updating CoreOS'){:target="_blank"}

Great, so with updates to the OS handled automatically, safely, and quickly, what else is of interest?

CoreOS is setup to run everything within a container, as I mentioned before there are very few other binaries installed on the system.  Even small system utilities like SNMP are run within a container.  This may seem a little odd at first but once again really makes maintenance over a cluster of these machines significantly easier, more secure, and more fluid.

### Configuration 

CoreOS allows for easy configuration through cloud config YAML files -- [Documentation](https://coreos.com/os/docs/latest/cloud-config.html 'Cloud config Docs'){:target:"_blank"}.  We are currently using CoreOS on premise with ESX so we create and use config-drive isos and mount them to each server.  Each time CoreOS boots it reads the config drive and loads the parameters specified.  This makes spinning up a new Bamboo Agent dead simple and extremely fast.  

When using cloud providers (such as AWS) these details can be added within the `User data` field when provisioning a new instance.  We use our cloud-config iso to 

* Add our internal CA 
* Start an SNMP Docker container
* Create and start a Bamboo data container
* Create and start a Bamboo agent container
* Create a Docker cleanup script on a timer

The best part about this type of configuration is that we can now take this and paste it into an AWS `User data` field and begin to expand our systems out into the cloud without having to worry about a different set of parameters.

Here is an example of how we accomplish some of these things:
{% gist c9a507c76ade421a6e14 %}

If you have read through the config file above, you may note I have disabled some of CoreOS's more interesting features, namely the key-value etcd store and the low-level orchestration tool fleetd.  For our builds we use CoreOS merely as the vessel to run our Docker containers on and nothing more.  We have no need for the extra features, nor do I foresee a time when they would bring added value to our build infrastructure.

### Wrap up

Utilizing CoreOS for our build infrastructure has significantly reduced our team's day to day administration of our build systems.  The built in configuration tools have allowed us to scale more quickly and easily than we have ever been able to in the past.