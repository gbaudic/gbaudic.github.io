---
layout: single
title: "Using Eclipse Oomph to produce a customized, portable Eclipse"
date: 2023-08-15 16:00:00 +0200
tags: linux oomph eclipse portable java groovy
---

## The issue

In a software development team, it is simpler when everybody uses the same tools... It turns out that I have been recently tasked to do such a thing: package a ready-to-use Eclipse IDE based on the latest version available and with some plugins included, so our client can easily develop scripts on its side to extend our product. 

## Solution

### Starting point: an existing Eclipse distribution

The file that we have after running the wizard is far from building a fully functional product... Wouldn't it be great if we were able to use an existing Eclipse distribution, for example, Eclipse IDE for Java developers, as our starting point? Turns out, it is possible. 

### Adding extra plugins

This one is pretty well documented in the Eclipse Oomph FAQ page. For convenience, I will repeat it here:

  1. 

### Editing the eclipse.ini file

Pretty straightforward. 

### Setting user preferences

### Running Oomph

We were supposed to get a functional product, but so far, all that we have is an XML file. It is now time to test it, and hopefully, turn it into our product. To do so, we will have to leave Eclipse for another component: the Oomph installer. 

### Making the result portable

By default, Oomph will make use of what is called a _Bundle pool_. Put simply, it is a common repository which, when using several Eclipse products on a single computer, allows to share the JAR files between them, thus saving disk space. This means that in the installation directory, there is no longer a ``plugins/`` directory: everything is located in the pool. This is clearly blocking to make our Eclipse installation portable.  
The solution to get back our ``plugins/`` directory is simply to **uncheck the checkbox in front of the Bundle pool line** on the first page of the Oomph setup wizard. You will also want to uncheck the "Launch installation" setting at the last page of the wizard, and archive your install into the distributable archive file **prior to** running it. This will ensure that any of the preferences changes shipped with your product actually get applied to your users when they start their packaged Eclipse. 

## Next step: automation

command-line Oomph plugin

## References

  * <https://www.eclipse.org/forums/index.php/t/1110408/>
  * <https://wiki.eclipse.org/Eclipse_Oomph_Authoring#The_preference_task_does_not_setup_any_preferences>