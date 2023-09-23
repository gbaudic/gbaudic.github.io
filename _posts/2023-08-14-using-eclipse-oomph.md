---
layout: single
title: "Using Eclipse Oomph to produce a customized, portable Eclipse"
date: 2023-08-14 20:00:00 +0200
tags: linux oomph eclipse portable java groovy installation
---

## The issue

In a software development team, it is simpler when everybody uses the same tools... It turns out that I have been recently tasked to do such a thing: package a ready-to-use Eclipse IDE based on the latest version available and with some plugins included, so our client can easily develop scripts on its side to extend our product. 

## Solution

To do so, I chose to rely on an Eclipse component which has been around for quite a few years now, and which also happens to be the default choice when downloading eclipse: the Oomph installer. This product relies on configuration to know what to install, in the form of ``.setup`` (technically XML) files. Eclipse provides an editor to easily author them, the Oomph Setup SDK. 

### Starting point: an existing Eclipse distribution

The file that we have after running the wizard (File > New > Oomph > Oomph Product model) is far from building a fully functional product... Wouldn't it be great if we were able to use an existing Eclipse distribution, for example, Eclipse IDE for Java developers, as our starting point? Turns out, it is possible.  
To do so, add a new child to your P2 Director task; choose the version to be 0.0.0, the type to be None, and the name is the package name (e.g., epp.package.java for Eclipse IDE for Java developers). To find the right name, either search for it on the Internet or, if you are already working with the Eclipse distribution you are willing to start from, just look for it under Help > About Eclipse IDE > Installation details in the Installed software tab.   
Be aware, though, that despite having put here the package name corresponding to the eclipse distribution you are willing to start from, this is not going to install the exact same plugin set as the one from the distribution downloaded from the eclipse website, but a more reduced one. You can however add the missing bits afterwards (see the next section). Note also that you may have to explicitly include some eclipse plugins in your setup if an extra plugin depends on them: in my case, I had to explicitly install the JDT, despite it being brought by ``epp.package.java``, because it was required by the Groovy plugin.  
I strongly recommend against putting any explicit version number here, especially if the version is already constrained by the P2 repository URL. 

### Adding extra plugins

This one is pretty well documented in the Eclipse Oomph FAQ page. For convenience, I will repeat it here:

  1. add a P2 Director child task to your *.setup, if it doesn't already have one (in fact, it probably already has one)
  2. add a Repository child in your P2 Director task
  3. in the Properties of the Repository, set the URL to a valid p2 repo (e.g., http://update.eclemma.org). This is the one you would have put into the "Install new software" URL field at the top of the dialog box. 
  4. right-click that Repository and choose Explore, to open the Repository Explorer view
  5. in the Repository Explorer view, change to [X] Compatible and (*) Minor at the lower-right hand corner. These options are located at the bottom right-hand corner of the view.
  6. drag & drop the version from the Repository Explorer view into the P2 Director task of your ``*.setup`` editor, to create a Requirement child. Now at this point, there are actually two different elements from the view that you can drag and drop: the name of the plugin, or the version of the plugin. Both will work, but they will produce a different result: the name will produce an entry without any version constraint (it reads 0.0.0 in the Version property), whereas the version will add an entry with a version constraint based on the current version and the selection you made before in the view (Minor, if you followed this blog post).  

### Editing the eclipse.ini file

Pretty straightforward. You can add a child for this, set the key and the value, and select if it is an Eclipse or a Java VM argument. For example, to make your new Eclipse start with 4 GB of RAM, use as key ``-Xmx``, value ``4096m``, and tick the box to indicate this is actually a VM parameter so that it lands at the right place in the file. 

### Setting user preferences

For this purpose, you will have to select another type in the Setup editor.  
However, this approach implies that you already know the name of the configuration option you are trying to set. Another way of doing it is to use in the toolbar the "Capture preferences" button, which allows you to use any of the configuration options set in your current IDE and add them to your setup file. In my case of a Java IDE, for example, I chose to have the heap status shown by default and to circumvent the Welcome screen on the first run.  
There is also a Resource Creation task which allows you to create a file and populate it with some values at once, but I have not had to use it so far.  
**Important**: the preferences are being setup on the first run of the installed IDE, not at installation time. Therefore, to ensure this, your custom IDE needs to have the ``org.eclipse.oomph.setup`` feature in it. I recommend adding it explicitly by looking for it in the Expert view of the Repository Explorer while browsing the eclipse P2 repository. 

#### Specific case: the JRE settings

Some research on the eclipse forums showed me that it was indeed not possible to setup preferences where the contents are presented in the form of an XML blob. Unfortunately, the *list* of available JREs falls into this category. Since this is not an Eclipse but rather an Eclipse **JDT** preference, both the eclipse where you are editing the ``.setup`` file and the installed one will require the Oomph Setup SDK to be installed, which brings the Oomph Setup JDT feature.  
Once this is done, you will be able to add as a new child to your file a JRE Task. The properties to set are pretty straightforward: pick a name, a version, select the Standard VM category, choose if this is supposed to be the default or not, and pick a path. By default, this is is going to be a variable, which means that the user will have to specify it when running Oomph, but it is perfectly possible to hardcode a path here as well if you know what you are doing. Beware though, that the path you are specifying will need to point to a valid JRE on the computer where the installation will take place: failure to do so will greet you with an error on the first launch otherwise. I also noticed that in this erroneous case, the other preferences (if any) would not be applied.  
Note that the most recent versions of eclipse ship with a JRE: it will already appear in the list without any action required. The JRE Task is therefore only required if you do not want (or cannot) use a Java 17 JRE on your projects. 

### Running Oomph

We were supposed to get a functional product, but so far, all that we have is an XML file. It is now time to test it, and hopefully, turn it into our product. To do so, we will have to leave Eclipse for another component: the Oomph installer. It has to be downloaded separately from eclipse.  
Note that if your goal is simply to make a custom eclipse for the rest of your development team, which you will be able to easily edit, upgrade and for which the target hosts are connected to the Internet, you can simply distribute your ``.setup`` file and stop your reading here. 

### Making the result portable

By default, Oomph will make use of what is called a _Bundle pool_. Put simply, it is a common repository which, when using several Eclipse products on a single computer, allows to share the JAR files between them, thus saving disk space. This means that in the installation directory, there is no longer a ``plugins/`` directory: everything is located in the pool. This is clearly blocking to make our Eclipse installation portable.  
The solution to get back our ``plugins/`` directory is simply to **uncheck the checkbox in front of the Bundle pool line** on the first page of the Oomph setup wizard. You will also want to uncheck the "Launch installation" setting at the last page of the wizard, and archive your install into the distributable archive file **prior to** running it. This will ensure that any of the preferences changes shipped with your product actually get applied to your users when they start their packaged Eclipse. 

## Next step: automation

The previous version of our eclipse-powered IDE got built at each run of our CI pipeline. Now with Oomph, we have to use a GUI, which is far less practical to automate. Fortunately, there is a command-line Oomph plugin which can be added to an existing eclipse installation. I have not been able to experiment with it for the moment, but it looks promising. 

It is worth noticing that Oomph can do way much more than what is explained in this post: in addition to plugins and configuration, it can also be instructed to setup your full workspace, including projects, Git repositories...

## References

  * <https://www.eclipse.org/forums/index.php/t/1110408/>
  * <https://www.eclipse.org/forums/index.php/t/1074416/>
  * <https://wiki.eclipse.org/Eclipse_Oomph_Authoring#The_preference_task_does_not_setup_any_preferences>
  * <https://wiki.eclipse.org/Eclipse_Installer>

