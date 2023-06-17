---
title: "Remember to read before waiting"
date: 2023-06-17 00:34:00 +0800
categories: Tech
---

# How it started
Recently at work due to certain complicated reasons, we are looking for a dockerless solution to build docker images for our mostly Java based applications. [Jib](https://github.com/GoogleContainerTools/jib) from Google is one strong candidate we are trying out.

> ## What is Jib?
>Jib builds optimized Docker and OCI images for your Java applications without a Docker daemon - and without deep mastery of Docker best-practices. It is available as plugins for Maven and Gradle and as a Java library.

It all works nicely until coming to one of our module that requires building with a relatively complex base image from the local docker daemon.

Jib does support building with reading base image from the local docker daemon. However, rather puzzlingly, it simply gets stuck at 25% saying 'Processing base image' for this particular base image.

![jib stucks](/assets/2023-06-17-Remember-to-read-before-waiting/246496811-db94355c-8d06-4540-a22d-a9d2f98510f6.png)

Even more mysteriously, it seems to gets stucked/unstucked seemingly arbitrarily if we remove/add some random instructions from/to the dockerfile used to build the base image and rebuild it. Such lines can be almost any line ranging from simply setting an environment variable to running a complicated command. 

At first we thought it might be due to the size of the base image or the number of files within the base image. Evetually neither was the case as even a base image built with [a simple dockerfile](https://gist.github.com/EternalWind/b177820578a316dbca1bdb099984ad47) that only sets environment variables could cause it to stuck.

![simple dockerfile](/assets/2023-06-17-Remember-to-read-before-waiting/Screenshot%202023-06-17%20163033.png)

It almost appears to suggest there is an implicit instruction count limit to the base image jib can use. And adding to the fun, different type of instruction weights differently.

# Root cause
After some diggings into jib's source code, I found out it's because of an implementation oversight when jib tries to inspect the base image summary read from the local docker daemon.

Jib creates a process running a ```docker inspect``` command and invokes ```Process.waitFor()``` immediately afterwards without reading from stdout. It only tries to read from stdout after waitFor() returns which only occurs when the process terminates.

![implementation issue](/assets/2023-06-17-Remember-to-read-before-waiting/Screenshot%202023-06-17%20164120.png)

As a result if the size of the process output to stdout is significant enough to overwhelm the buffer, the process would become pending until someone read from stdout to clear out the buffer. This results in a deadlock situation that makes waitFor() never returns.

# The fix
Well... The fix itself is rather simple. Just continually read from the process's output while the process runs before calling waitFor() on the process. 

Until Google merges my [PR](https://github.com/GoogleContainerTools/jib/pull/4048) to fix it and release a new version of jib, you will need to build jib from source with the necessary changes proposed in the PR yourself.