---
layout: single
title: "Calling batch commands from Cygwin"
date: 2022-11-02 16:00:00 +0100
tags: windows linux cygwin shell cmd batch
---

I am currently working on a product developed in Java. To make it more convenient to users, we also ship some command-line scripts with it to start and stop its different components. Versions for Linux (``.sh``) and Windows (``.cmd``, aka batch files) are consequently provided. The database component of the product has, unlike the others, an unpredictable and often long shutdown time. However, we had to guarantee the proper shutdown of this component, which means that after some waiting time we have to kill it.

## The issue

I decided to use the timeout command to wait for a sign of proper shutdown of the database, in which case the following ``kill`` command is not necessary. While it ran fine on my Windows system, it kept spilling out errors on the client machine: 

    timeout: unsupported command /t

## Solution

It turns out that the cygwin command was being used, despite the script being a ``.cmd``. This was possible because the cmd was being run from a bash script launched in Cygwin, and in the PATH variable, Cygwin binaries have precedence over the Windows ones. There are two possible solutions to this issue. 

  * Use the Linux syntax instead. This was not an option because the script has to be able to run from a pure Windows console window too. 
  * Ensure the Windows command gets used all the time. Simply wrapping the call to timeout in a cmd call did not work, so I eventually managed to get the correct command by specifying the full path to it: ``timeout`` became ``c:\Windows\system32\timeout``. The full line then becomes

    cmd /c "c:\Windows\system32\timeout /1 /nobreak > nul"


## References

  * <https://sourceware.org/legacy-ml/cygwin/2018-02/msg00209.html>
  * <https://superuser.com/questions/1172840/run-a-windows-command-from-cygwin>
