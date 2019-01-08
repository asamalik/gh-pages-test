---
id: 415
title: Building Modules for Fedora 27
date: 2017-09-18T08:34:02+00:00
author: adam
guid: http://blog.samalik.com/?p=415
permalink: /building-modules-for-fedora-27/
catchflames-sidebarlayout:
  - default
dsq_thread_id:
  - "6181846925"
categories:
  - Fedora
  - Modularity
---
Let me start with a wrong presumption that you have everything set up &#8211; you are a packager who knows what they want to achieve, you have a dist-git repository created, you have all the tooling installed. And of course, you know what Modularity is, and how and why do we use modulemd to define modular content. You know what Host, Platform, and Bootstrap modules are and how to use them.

Why would I make wrong presumptions like that? First of all, it might not be wrong at all. Especially, if you are a returning visitor, you don&#8217;t want to learn about the one-off things every time. I want to start with the stuff you will be doing on regular basis. And instead of explaining the basics over and over again, I will just point you to the right places that will help you solve a specific problem, like not having a dist-git repository.

## Updating a module

Let&#8217;s go through the whole process of updating a module. I will introduce some build failures on purpose, so you know how to deal with them should they happen to you.

Start with cloning the module dist-git repository to get the modulemd file defining the installer module.

<pre>$ fedpkg clone modules/installer
$ cd installer
$ ls
Makefile installer.yaml sources tests</pre>

I want to build an updated version of installer. I have generated a new, updated version of modulemd (I will be talking about generating modulemd files later in this post), and I am ready to build it.

<pre>$ git add installer.yaml
$ git commit -m "building a new version"
$ git push
$ mbs-build submit
...
asamalik's build #942 of installer-master has been submitted</pre>

Now I want to watch the build and see how it goes.

<pre>$ mbs-build watch 942
Failed:
 NetworkManager https://koji.fedoraproject.org/koji/taskinfo?taskID=21857852
 realmd https://koji.fedoraproject.org/koji/taskinfo?taskID=21857947
 python-urllib3 https://koji.fedoraproject.org/koji/taskinfo?taskID=21857955

Summary:
 40 components in the COMPLETE state
 3 components in the FAILED state
asamalik's build #942 of installer-master is in the "failed" state</pre>

Oh no, it failed! I reviewed the Koji logs using the links above, and I see that:

  * NetworkManager and python-urllib3 failed randomly on tests. That happens sometimes and just resubmitting them would fix it.
  * realmd however needs to be fixed before I can proceed.

After fixing realmd and updating the modulemd to reference the fix, I can submit a new build.

<pre>$ git add installer.yaml
$ git commit -m "use the right version of automake with realmd"
$ git push
$ mbs-build submit
...
asamalik's build #942 of installer-master has been submitted</pre>

To watch the build, I use the following command.

<pre>$ mbs-build watch 943
Failed:
 sssd https://koji.fedoraproject.org/koji/taskinfo?taskID=21859852

Summary:
 42 components in the COMPLETE state
 1 components in the FAILED state
asamalik's build #943 of installer-master is in the "failed" state</pre>

Good news is that realmd worked this time. However, sssd failed. I know it built before, and by investigating the logs I found out it&#8217;s one of the random test failures again. Resubmitting the build will fix it. In this case, I don&#8217;t need to create a new version, just resubmit the current one.

<pre>$ mbs-build submit
...
asamalik's build #943 of installer-master has been resubmitted</pre>

Watch the build:

<pre>$ mbs-build watch 943
Summary:
 43 components in the COMPLETE state
asamalik's build #943 of installer-master is in the "ready" state</pre>

### Rebuilding against new base

The Platform module has been updated and there is a soname bump in rpmlib. I need to rebuild the module against the new platform. However, I&#8217;m not changing anything in the modulemd. I know that builds map 1:1 to git commits, so I need to create an empty commit and submit the build.

<pre>$ git commit --allow-empty -m "rebuild against new platform"
$ git push
$ mbs-build submit
...
asamalik's build #952 of installer-master has been submitted</pre>

## Making sure you have everything

Now it&#8217;s the time to make sure you are not missing a step in your module building journey! Did I miss something? Ask for it in the comments or <a href="https://twitter.com/adsamalik" target="_blank">tweet at me</a> and I will try to update the post.

### How do I get a dist-git repository?

To get a dist-git repository, you need to have your modulemd to go through a <a href="https://fedoraproject.org/wiki/Module:Review_Process" target="_blank">Fedora review process for modules</a>. Please make sure your modulemd comply with the <a href="https://fedoraproject.org/wiki/Module:Guidelines" target="_blank">Fedora packaging guidelines for modules</a>. Completing the review process wil result in having a dist-git repository.

### What packages go into a module?

Your module need to run on top of the <a href="https://fedoraproject.org/wiki/Host_and_Platform" target="_blank">Platform module</a> which together with the Host and Shim modules form the base operating system. You can see the <a href="https://github.com/fedora-modularity/hp" target="_blank">definition of Platform, Host, and Shim</a> modules on github.

You should include all the dependencies needed for your module to run on the Platform module, with few exceptions:

  * If your module needs a common runtime (like Perl or Python) that are already modularized, you shoud use these as a dependencies rather than bundling them to your module.
  * If you see that your module needs a common package (like net-tools), you shouldn&#8217;t bundle them either. They should be split into individual modules.
  * To make sure your module doesn&#8217;t conflict with other modules, it can&#8217;t contain the same packages as other modules. Your module can either depend on these other modules, or such packages can be split into new modules.

To make it easier during the transition from a traditional release to a modular one, there is a set of useful scripts in the <a href="https://github.com/fedora-modularity/dependency-report-scripts" target="_blank">dependency-report-scripts</a> repository, and the results are being pushed to the <a href="https://github.com/fedora-modularity/dependency-report" target="_blank">dependency-report</a> repository. You are welcome to participate.

#### Adding your module to the dependency-report

The module definitions are in the <a href="https://github.com/modularity-modules/" target="_blank">modularity-modules</a> organizations, in a README.md format simlar to the <a href="https://github.com/fedora-modularity/hp" target="_blank">Platform and Host definition</a>. The scripts will take these repositories as an input together with the Fedora Everything repository to produce dependency reports and basic modulemd files.

### How can I generate a modulemd file?

The <a href="https://github.com/fedora-modularity/dependency-report-scripts" target="_blank">dependency-report-scripts</a> can produce a basic modulemd, stored in the <a href="https://github.com/fedora-modularity/dependency-report" target="_blank">dependency-report</a> repository. The modulemd files stored in the dependency-report repository reference their components using the f27 branch for now. To produce a more reproducible and predictable result, I recommend you to use the generate\_modulemd\_with_hashes.sh script inside the dependency-report-scripts repository:

<pre>$ generate_modulemd_with_hashes.sh MODULE_NAME</pre>

The result will be stored at the same place as the previous modulemd, that was using branches as references.

<pre>$ ls output/modules/MODULE_NAME/MODULE_NAME.yaml</pre>

### Where do I get the mbs-build tool?

Install the module-build-service package from Fedora repositories:

<pre>$ sudo dnf install module-build-service</pre>

### What can I build to help?

See the [F27 Content Tracking repository](https://github.com/fedora-modularity/f27-content-tracking) and help with the modules that are listed, or propose new ones. You can also ask on #fedora-modularity IRC channel.

### Where can I learn more about Modularity?

Please see the <a href="https://docs.pagure.org/modularity" target="_blank">Fedora Modularity documentation website</a>.

&nbsp;

&nbsp;
