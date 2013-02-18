---
layout: post
title: Building my own CyanogenMod image
---

### {{ page.title }}

I started attempting to build CyanogenMod on my OSX MBP - but it quickly got complex and _experimental_. So I opted for a different approach - I'd build the image in VM. BUT to make this easier to replicate I built a vagrant config.

[Vagrant](http://www.vagrantup.com/) is a build/testing tool that automates VM setup. I use [homebrew](http://mxcl.github.com/homebrew/) so installing vagrant was super easy:

    brew install vagrant

Now download the base ubuntu image I used

    vagrant box add precise64 http://files.vagrantup.com/precise64.box

Grab a copy of my vagrant config and start up the VM:

    git clone https://github.com/farproc/CyanogenModBuildEnv.git
    cd CyanogenModBuildEnv
    vagrant up

A copy of the ubuntu VM should be spawned and configured with all the required packages to successfully build CyanogenMod. Then it's just a matter of following the CyanogenMod guide [itself](http://wiki.cyanogenmod.org/w/Build_for_crespo) (this is the nexus-s steps, but all the others are there too).

At one point in the build you need to provide _proprietary blobs_ directly from the phone (via adb) - you could install USB support for VirtualBox, so you could forward the _adb server_ port into our VM from your host. A nice little trick that saves a bit of time:

    # make sure the VM isn't already running its own adb server
    vagrant ssh
    sudo su -
    adb kill-server
    exit
    exit

    # start on the host and tunnel
    adb devices
    ssh -nNR127.0.0.1:5037:127.0.0.1:5037 -oExitOnForwardFailure=yes -p 2222 -i ~/.vagrant.d/insecure_private_key -l vagrant localhost

Now you can issue adb command from within the VM (or run the extract-files.sh script).


Hope that helps...
