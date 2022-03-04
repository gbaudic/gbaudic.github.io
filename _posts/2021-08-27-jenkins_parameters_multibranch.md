---
layout: single
title: "Jenkins build parameters and Multibranch pipelines"
date: 2021-08-27 14:00:00 +0200
tags: tips-and-tricks jenkins continuous-integration development multibranch
---

It took me almost an afternoon of trial-and-error, googling and fixing to get there, which is (for me, at least) way too much for such a basic task. So I thought I would sum it up in a blog post if it can save someone some time in the future... 

## The Issue

In a project I was working on, I had two branches on my Git repository: ``develop``, on which (you guessed it) everyday development ~~pain~~ efforts happen, and ``master``, which only gets updated for each release. Now, for some reason, the master branch had not been updated in a while, and some releases had been missed: this means I had to trigger Jenkins builds manually on my ``master`` branch. Sounds easy enough, given that I already have a Jenkinsfile and a working pipeline. Really? 

At first try, this simply did not work: regardless of the branch which was updated, my pipeline kept building the ``develop`` branch. Lifting the restrictions on the branches that could be built (by putting ``*/*`` in the list instead of the ``\*\*`` Jenkins will happily put if you leave the field blank, which is a known Jenkins bug by the way) did not help. 

## Multibranch pipelines

Fortunately, Jenkins has a feature for this very purpose: __Multibranch pipelines__. This kind of pipeline will analyze a repository at a regular time interval (or after a trigger from the SCM), and for each branch containing a Jenkinsfile, create a pipeline based on it and run it. 

Creation is straightforward: pick a name, a repository to build from and the authentication details to access it, and you are good to go. You can even start from an existing single-branch pipeline, which is not what I did there despite having one available. 

Good, now I can build ``master`` and ``develop`` separately, and only when the right branch changes. But there is an issue now...

## Where are my parameters?

In my initial pipeline, I had specified a parameter to exclude testing from the build, in an attempt to speed it up. But looking at the web UI, the checkbox "This build has parameters" is missing in Multibranch pipelines... So for the moment, my job fails all the time :-(

## Solution

It turns out, there is actually a way to specify parameters for a Multibranch pipeline: they have to be in the Jenkinsfile. You can then access them in the remainder of the script using the syntax ``${params.PARAMETER_NAME}``, and they will appear in the web UI when you try to trigger a job manually. A snippet can then be (I use the Declarative pipeline syntax here): 

    pipeline {
        agent any
        parameters {
            booleanParam(name: 'WITH_TESTS', defaultValue: true, description: 'Run the tests')
        }
        stages {
            stage('Build') {
                steps {
                    sh './build.sh'
                }
            }
            ...
        }
    }

I honestly do not know what happens if you do not specify a default value above when the pipeline gets run automatically. In doubt, just put one. 

## References

  * [Official Jenkins documentation on parameters](https://www.jenkins.io/doc/book/pipeline/syntax/#parameters)
  * [Original blog post](https://shenxianpeng.github.io/2021/03/jenkins-dynamic-default-parameters/)
