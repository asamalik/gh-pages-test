---
id: 425
title: 'Module builds &#8211; how do they work?'
date: 2017-09-25T11:49:06+00:00
author: adam
layout: post
guid: http://blog.samalik.com/?p=425
permalink: /module-builds-how-do-they-work/
catchflames-sidebarlayout:
  - default
dsq_thread_id:
  - "6181828431"
categories:
  - Fedora
  - Modularity
---
Have you ever wondered how modular builds work? In my previous post called <a href="http://blog.samalik.com/building-modules-for-fedora-27/" target="_blank">Building Modules for the F27 Server</a>, I described the steps a packager needs to go through in order to produce a successful build in the Fedora infrastructure. But what exactly is going on at the backstage?

[<img class="aligncenter size-large wp-image-427" src="https://blog-shaman.rhcloud.com/wp-content/uploads/2017/09/factory-1024x512.png" alt="factory" width="600" height="300" />](https://blog-shaman.rhcloud.com/wp-content/uploads/2017/09/factory.png)

The build pipeline sees modules as build recipes for RPM packages. Each module has a list of SRPM (source RPM) packages that get build within the module. These packages are divided into build groups which are processed in batches one by one. Build groups are useful in situations when certain packages are needed in the buildroot to build some other ones within the same module. How these build groups work and what services are they part of?

All packages are build in <a href="https://pagure.io/koji/" target="_blank">Koji</a>—the same service that builds packages for the traditional Fedora release. Interesting is that Koji doesn’t know anything about modules or build groups. And it uses the same mechanics in both modular and traditional releases. How it knows what to do? That’s where the <a href="https://pagure.io/fm-orchestrator" target="_blank">Module Build Service (MBS)</a> comes in.

MBS orchestrates package builds in Koji. This service makes sure that packages are built in a defined order by leveraging the concept of build groups, and reports the overall state of module builds to the user and other services. While MBS manages most of the steps of module builds, it relies on Koji to do the actual building, and to store the output such as binary packages and logs.

When a module build is finished, it is processed by other services to produce the final artifacts such as RPM repositories, container base images, cloud images, or bootable iso images. Artifacts are typically composed out of multiple modules.

## Building a module

Everything starts when a module build gets submitted to MBS. At the time of writing this post, this action is done manually by the packager. They need to run the \`mbs-build submit\`command in their dist-git repository to get the build submitted. However, a new service called <a href="https://fedoraproject.org/wiki/Infrastructure/Factory2/Focus/Freshmaker" target="_blank">Freshmaker</a> is being deployed. This service will automatically submit builds for every commit to dist-git.

### Initial check

When a build is submitted to MBS, it enters the &#8220;init&#8221; state. During this state, MBS checks if the modulemd file is valid, and that all components of the modules are available. If everything is alright, the state switches to &#8220;wait&#8221;.

Modules in the &#8220;wait&#8221; state are waiting for a capacity in the build system. When a capacity is available, MBS switches the state to &#8220;build&#8221;. And that&#8217;s where the real building starts.

### Preparing a buildroot

Before diving into describing the &#8220;build&#8221; state, we need to understand two concepts Koji uses during module builds: tags and tag inheritance. Tags are basically groups of Koji builds. A single build can be tagged into multiple tags. Tags can represent buildroots, releases, or any other groups of builds—like modules. MBS creates two tags for every module build: the buildroot, and the result. Buildroots are typically composed of multiple modules. Koji uses the tag inheritance mechanism to combine multiple tags into one. How is a module buildroot created?

Module buildroot is a tag in Koji representing the environment—a set of packages—where a module gets build. Constructing a buildroot is a first step of the &#8220;build&#8221; state. The buildroot is an empty tag that inherits all the modules specified as build dependencies in the modulemd that is being built. When this is done, MBS can submit package builds into Koji.

The first package that is built as a part of every module build is the \`module-build-macros\`. It contains a set of macros that are used when building other packages. When this build is finished, the result is tagged back to the buildroot tag—so it is available for all other components.

### Building packages in build groups

Packages are built in batches called build groups. These build groups are defined in modulemd using a buildorder value. The buildorder value is an integer, every component (package) has it, and the default value is zero. Components with the same buildorder value are part of the same build group.

MBS schedules build groups one-by-one, starting with the lowest buildorder value. All packages within a build group are submitted at once and built in parallel. When all builds within the build group finish, the results are tagged back to the buildroot tag—so they are available for the next ones. This is repeated until all build groups are finished.

When all build groups are finished, MBS changes the module build state. If all component builds succeeded, the build state is set to &#8220;done&#8221;. In case there is one or more failures, the state is set to &#8220;failed&#8221;. Both of these states mean that the build phase has ended.

### After the build

There is another state in MBS indicating that modules are ready to be part of a larger compose and consumed by the end-user. The state is called &#8220;ready&#8221;. Right now, MBS switches all states &#8220;done&#8221; to &#8220;ready&#8221; automatically. However, in the future, this will be probably switched by an external service that sees the bigger picture. Part of the bigger picture can be rebuilding all modular dependencies, additional integration testing, etc.

## Next step &#8211; release

Modules in the &#8220;ready&#8221; state are not automatically available to the end user. To have a certain module available in a release, it needs to be added to a compose which is then used to produce the final artifacts such as RPM repositories, container base images, cloud images, or bootable iso images. I might write additional blog post describing this in the future.

This post has been mostly focused on the build phase. There are many other interesting services in the Factory 2.0 that make Modularity possible. Have a look at the <a href="https://fedoraproject.org/wiki/Infrastructure/Factory2/Focus" target="_blank">Factory 2.0 Focus Documents</a> for more information. You can also learn <a href="http://blog.samalik.com/building-modules-for-fedora-27/" target="_blank">how to build modules for the F27 Server</a>, see the <a href="https://github.com/fedora-modularity/f27-content-tracking" target="_blank">F27 content plan</a> and help us with building more modules, or browse the <a href="https://docs.pagure.org/modularity" target="_blank">Modularity documentation</a> to get more information about the effort.