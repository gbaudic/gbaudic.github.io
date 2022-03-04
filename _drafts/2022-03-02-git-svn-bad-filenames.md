---
layout: single
title: "SVN to Git migration on Windows: when filenames attack"
date: 2022-03-02 16:00:00 +0100
tags: git svn windows linux migration tips-and-tricks
---

When it comes to version control software, SVN used to be a hugely popular choice until it got superseded by Git. This means that from time to time, for example when a project's ownership changes from one team to another, you get tasked with moving the code repository from SVN to Git as part of the migration. Such a move is so unlikely that Git already offers an out-of-the-box solution to achieve this: git svn. 

In our case, the previous team had decided to give us the contents of their SVN __server__, which ran on Linux. On our side, difficulties to manage to get specific versions of Git and SVN that were compatible with the repository format meant that we were working on Windows. There was only a small issue for us: for many of the repositories we had to move, it simply DID NOT WORK. More precisely, after a while, we were faced with a message like this one:

  fatal: Unable to create 'C:/Users/user1/migrate/proj1/.git\svn\refs\remotes\https;C:\Program Files\Git\index.lock': 
  Invalid argument write-tree: command returned error:128

followed by a list of files Git was trying to create for this specific revision. Notice the completely messed-up file path for the index lock file. 

## Filenames MATTER.

A quick look at the list of files Git was trying to write for the revision should have led us to spot the issue: some of them contained a character which is illegal for filenames on Windows, namely ``:``. On Linux, this is harmless, but on Windows, this character can only be used to separate the drive name from the rest of the path. Fortunately for us, these were only used to create a few tags. 

## Solution

At this point, there are two solutions:
  * **Stop doing it on Windows and use a Linux box instead**. This does not require any instructions, but was apparently not possible in our case due to versions incompatibilities between the SVN client we had and the SVN server which was used to host the data on the other team's side. 
  * **Keep doing it on Windows and skip the offending revisions**.
  
The second approach requires you to go through the whole history of the SVN repository you are trying to convert and spot the numbers of the offending revisions. I was too lazy to try to write a bash snippet and decided instead to browse the history using TortoiseSVN... but this can be tedious. Another lazy approach can be to just try, wait for the first error, and exclude the revisions as they show up. However, I would really recommend to do things properly upfront by analyzing some SVN output first, since this will save you a lot of time. Especially if the repository tends to be big.   
Once this is done, you can make a ``git svn clone`` up to the first offending revision (excluded), followed by as many ``git svn fetch`` as needed, depending on how many erroneous versions you have (and hence, the number of version ranges it translates to). The option ``-r x:y`` will help you there (with x and y the first and last version requested, respectively; use ``HEAD`` to denote the last version to date). 

## Are we done yet?

Once these operations are complete, you may want to check that things ran correctly by issuing a ``git svn info``. If you correctly followed the advice above, the result will surprise you: the repository is still in the state it was after the first ``clone`` command! Fear not, as the next commands were actually executed. To make the repository state accurately reflect them, just run ``git svn pull``. Voilà! Your converted repo is now ready to push... wait, is it?

In fact, it is not: the branches and tags from SVN have only be created as remote ones. If you want them pushed to your new git repository, they have to be local ones as well. This is actually a three-step approach:
  * Make all branches local
  * Make all tags local
  * Remove the SVN tags. Warning: only do this if your move to Git is definitive and you will not be using SVN again! 
  
To be fair, Atlassian provides a JAR file to perform these operations in their tutorials for their Bitbucket service. But if like me, you are not confident in running an untrusted Java binary (despite in this case the source code being available), you can achieve the same result using the good old terminal. 

The corresponding bash code looks like this (read it carefully before because it may need some adaptations in your case):

  # Create local branches
  git for-each-ref refs/remotes/svn --format="%(refname:short)" | sed 's#svn/##' | grep -v '^tags'| while read aBranch; 
  do 
    git branch "$aBranch" "svn/$aBranch" 
  done
  # Create the local trunk branch
  git branch -d trunk 
  # Create tags
  git for-each-ref refs/remotes/svn/tags --format="%(refname:short)" | sed 's#svn/tags/##' | while read aTag; 
  do
    git tag "$aTag" "svn/tags/$aTag"
  done
  # Delete remote tags (only if a definitive move to git is planned!!!)
  git for-each-ref refs/remotes/svn/tags --format="%(refname:short)" | grep 'svn/tags' | while read aTag; 
  do
    git tag -d "$aTag"
  done

Now all you have to do is push your repo and its tags to its new Git home!

## References

  * [StackOverflow post which solved the issue](https://stackoverflow.com/questions/63043265/index-lock-invalid-argument)
  * git-svn reference page: 
