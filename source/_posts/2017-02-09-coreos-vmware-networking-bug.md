---
layout: post
title: "Coreos VMWare Networking Bug"
permalink: coreos-vmware-networking-bug
date: 2017-02-09 14:17:30
comments: true
description: "Coreos VMWare Networking Bug"
keywords: "CoreOS, VMWare"
categories: CoreOS, VMWare

tags: CoreOS, VMWare

---

### Preface

Several weeks ago we (DevOps) began getting reports and tracking a number of seemingly unrelated build failures due to what appeared to be either a loss of network connectivity or general DNS lookup failures.  As per my previous posts, our build environment relies heavily on numerous different containers spinning up for each build and its associated test suites.  This model utilizes Docker's built-in bridge networking to create a new virtual ethernet adapter and NAT ports for communication to and from outside of the container.
Any new container spun up during a build is inherently ephemeral and gets automatically removed at the end of a build run whether the build was successful or not so the host is clean and ready for the next build.  With how many builds go through our servers a day the servers are constantly creating and destroying many containers, making this issue problematic to track down.

### Replicate

As the issues appeared seemingly randomly I decided to create a simple Bash one-liner to run a loop, creating a simple bash container, ping something, and then either exiting on success or dropping into a bash shell within the container on failure.  This allowed for not only reliably replicating the issue, but the ability to see that once in a failed state the container would not recover on its own.
{% highlight bash %}
for ((i=0; i<100; i++)) ; do docker run -it --rm bash bash -c "ping -c 1 google.com &> /dev/null && echo 'Success' || bash" ; done
{% endhighlight %}

### Resolution

Upon creating several of these containers and utilizing Systemd's networkctl tool on the CoreOS system itself, a pattern emerged.  The newly created virtual ethernet adapter was stuck in "configuring" and would never complete, thus unable to ping even its own gateway ip.  Digging further the `networkctl status ens192` command revealed there was a network file at the highest priority lexical ordering controlling the main ethernet interface `/run/systemd/network/00-.network`.  This particular file was not a part of my user defined cloud-config.  The file had no defined `[Match]` section and so was proceeding to control any new interface including Docker's virtual ones.  Coreos' virtual interfaces are by default unmanaged by Systemd networkd due to upstream issues including but not limited to the ones we were experiencing.  

The fix was as simple as adding a new unit file to start within our cloud-config to remove `/run/systemd/network/00-.network` at startup and restart systemd-networkd.  After applying the fix to all of our servers we are not seeing this issue, once again getting infrastructure out of the way of green builds.  

I ended up filing a bug report with CoreOS as this configuration comes from our use of the VMWare OVA and is created outside of our control.  The bug issue can be found over on CoreOS' GitHub: [Bug 1802](https://github.com/coreos/bugs/issues/1802 "Bug 1802"){:target='_blank'}