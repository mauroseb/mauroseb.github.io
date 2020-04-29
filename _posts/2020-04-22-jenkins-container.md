---
layout: posts
title: "Setup Container for Jenkins"
date: 2020-04-22
categories: [blog]
tags: [ containers, jenkins ]
author: mauroseb
excerpt_separator: "<!-- more -->"
---

## Quickly setup Jenkins

<!-- more -->
I have a Jenkins master running in my lab as a static container that I use to deploy the applications I would like to test.
Plenty of sources to achieve the same, but main is the official one[^1].

1. Pull the latest official image for Jenkins
<!-- more -->

{% highlight shell %}

        # podman pull jenkins/jenkins
        Trying to pull jenkins/jenkins...
        Getting image source signatures
        Copying blob 1607093a898c done
        Copying blob d4eee24d4dac done
        Copying blob c58988e753d7 done
        Copying blob 794a04897db9 done
        Copying blob 9a8ea045c926 done
        Copying blob 55cbf04beb70 done
        Copying blob 70fcfa476f73 done
        Copying blob 0539c80a02be done
        Copying blob 54fefc6dcf80 done
        Copying blob 911bc90e47a8 done
        Copying blob 38430d93efed done
        Copying blob 7e46ccda148a done
        Copying blob c0cbcb5ac747 done
        Copying blob 35ade7a86a8e done
        Copying blob aa433a6a56b1 done
        Copying blob 841c1dd38d62 done
        Copying blob b865dcb08714 done
        Copying blob 5a3779030005 done
        Copying blob 12b47c68955c done
        Copying blob 1322ea3e7bfd done
        Copying config cd14cecfdb done
        Writing manifest to image destination
        Storing signatures
        cd14cecfdb3a657ba7d05bea026e7ac8b9abafc6e5c66253ab327c7211fa6281

{% endhighlight %}


2. Create a directory to use as persistent volume for Jenkins data (also ensure it matches the contianer jenkins user UID: 1000)

        # mkdir -m 0022 /home/jenkins
        # chown -R 1000:1000 /home/jenkins


3. Test run

       # podman run --name jenkins-master --rm -d -p 8080:8080 -p 50000:50000 -v /home/jenkins:/var/jenkins_home:z jenkins/jenkins
       3f4a74966f26e4d98ac7daacf7f39f1a43f80388ce40adc7dfd067d0beb01f30

       # podman ps
       CONTAINER ID  IMAGE                             COMMAND  CREATED             STATUS                 PORTS                                             NAMES
       3f4a74966f26  docker.io/jenkins/jenkins:latest           About a minute ago  Up About a minute ago  0.0.0.0:8070->8080/tcp, 0.0.0.0:50000->50000/tcp  jenkins-master

4. Check podman logs for the admin password to login to the webUI


That's it!
Jenkins now is up and running time to tweak it and add your pipelines in it.
More on that in a future post.

## References

[^1]: [Jenkins.io - Installint](https://jenkins.io/doc/book/installing/)
