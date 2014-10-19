---
layout: post
title:  "Get up and running with Go Lang using Vagrant"
date:   2014-10-18
categories: go_lang
---
I have started to do some work with [Go lang](http://golang.org) recently and having sevral different `GOPATH`:s is a bit messy. So I decided to look into having a virtual dev environment.

I have used [Vagrant](https://www.vagrantup.com/) before and enjoy working with it so I created a [Vagrantfile](https://github.com/pecke01/go_vagrant/blob/master/Vagrantfile) for it that makes it possible to have different setups of go projects but be isolated from eachother.

How to use it:
* Install Vagrant
* Copy it to your local folder that you want to use for your GOPATH.
* Run `vagrant up` to setup the virtual machine including installing GO and setting the correct environmental variables.
* Use `vagrant ssh` to ssh into the machine
* Doing `cd $GOPATH` will take you to the shared folder with your host machine.
* Start coding

I decided to go this way since I like to use my local machine where I have all my editors and helpers setup just the way I like it. Hope that it will help you and feel free to change it anyway you like.
