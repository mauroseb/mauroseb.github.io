---
layout: posts
title: "Testing Openstack Patches"
date: 2020-04-29
categories: [blog]
tags: [ openstack, development ]
author: mauroseb
excerpt_separator: "<!-- more -->"
---

## Intro

This is the first post that regards to OpenStack development.
When you start the journey of collaborating in OpenStack upstream projects one of the ways to start helping out and learn at the same time is to do code reviews and testing.

This activity is not only paramount for the community but also will position you better when you start to submit patches. The more reviews you have, the more trustworthy you become.

OpenStack projects use **gerrit** for this purpose (in essence similar to gitlab, bitbucket or reviewboard). This is an excellent tool that integrates into git and allows to submit commits for review, scoring and many other features. You can find all documentation you need in the _OpenStack Contributor Guide_ and if time permits I will add some post on that too[^1] [^2] [^3]. Also check this [link](https://julien.danjou.info/rant-about-github-pull-request-workflow-implementation/) for a good reference that shows why many developers consider gerrit superior to the PR workflow of Github or Gitlab that most are used to. I guess for me, I started with gerrit first and it feels a lot more natural way to work.

But that is not what I am going to write about today. I just want to showcase how can you quickly test a patch in your own environment that has not been merged into the master branch yet, with an example.

Let's say you run into a bug and you look in bugzilla.redhat.com, bugs.launchpad.net/openstack/ or storyoard.openstack.org and you found someone already reported it. And even better there is a patch for it yet to be merged, but there is no package or contianer image you can download and install.
Then you could do the community and yourself a favor and test it.

Since we are dealing mostly with python code one could just find the package code in the test environment and apply the patch.

Following is an example for some Director (TripleO) code.
<!-- more -->

## The Problem

In my case the bug I ran into in this example is the following:
OpenStack 16 downloading container images from registry.redhat.io times out after the Bearer Token expires and starts to return **401. Unauthorized** for any subsequent requests. Since that is a known issue someone has kindly filed a bugzilla ticket for me:
 - <https://bugzilla.redhat.com/show_bug.cgi?id=1811798>


## The Patch

There is a patch that was merged some weeks ago but still there is no downstream package or image I can use in my deployment:
 - <https://review.opendev.org/#/c/713923/>

You can notice very useful information in the gerrit dashboard like: the description, the code, if it has been reviewed, votes, comments, CI gates passing, that it has been merged already.


## The Test

I log in into my undercloud server, change to the directory where the code for the project in question **tripleo-common** sits. Then download the patch. Notice the URL is slightly different than before so I can get the mime-encoded version of it. And apply it.

{% highlight console %}

[root@undercloud-osp16 ]# cd /usr/lib/python3.6/site-packages/
[root@undercloud-osp16 site-packages]# curl -4sSL https://review.opendev.org/changes/713923/revisions/current/patch?download | base64 -d | patch -p1
patching file tripleo_common/image/image_export.py
Hunk #1 FAILED at 114.
1 out of 1 hunk FAILED -- saving rejects to file tripleo_common/image/image_export.py.rej
patching file tripleo_common/image/image_uploader.py
Hunk #1 succeeded at 539 (offset -4 lines).
Hunk #2 succeeded at 551 (offset -4 lines).
Hunk #3 succeeded at 1727 (offset -4 lines).

{% endhighlight %}

In this case it complains that a hunk failed. A hunk is a segment of code in the diff formatted patch that clearly could not be applied for some reason. That reason is the irreconciliable difference between what the patch expects and what is in the code, like the fact that the downstream TripleO version I am using is a bit older. Some code could be refactored and moved elsewhere and if you need the change you would need to locate other lines or files manually where the code used to sit and apply the change manually.

I check this part of code is exception handling and it is already halfly managed by the next code block so I consider can live without it:

{% highlight console %}

 [root@undercloud-osp16 site-packages]# cat tripleo_common/image/image_export.py.rej
--- tripleo_common/image/image_export.py
+++ tripleo_common/image/image_export.py
@@ -114,6 +114,13 @@
         LOG.error(memory_error)
         remove_layer(image, blob_path)
         raise MemoryError(memory_error)
+    except requests.exceptions.HTTPError as e:
+        # catch http errors seperately as those can be retried in
+        # the image uploader
+        http_error = '[{}] HTTP error: {}'.format(image, str(e))
+        LOG.error(http_error)
+        remove_layer(image, blob_path)
+        raise
     except Exception as e:
         write_error = '[{}] Write Failure: {}'.format(image, str(e))
         LOG.error(write_error)
         
{% endhighlight %}

My next attempt to deploy the environment works like a charm and does not bail out while downloading the container images.

I could add my ```+1``` (good review) in gerrit and leave a comment of my test if there is some remark. Or if it went wrong, explain what happened and leave a negative review.  The patch in question, however, is already merged so no need for that.

Lastly since this is python the OpenStack community uses **tox** [^4] as standard way to test the code. You can run this tests in your local branch if you like.

{% highlight console %}
$ tox -e py36
$ tox -e pep8
$ tox -e functional
{% endhighlight %}

That's all.

## References

[^1]: [Setup and Learn GIT](https://docs.openstack.org/contributors/common/git.html)
[^2]: [Setup Gerrit](https://docs.openstack.org/contributors/common/setup-gerrit.html)
[^3]: [How to become a patch Guru ?](https://docs.openstack.org/contributors/code-and-documentation/patch-best-practices.html)
[^4]: [Tox](https://tox.readthedocs.io/en/latest/)
