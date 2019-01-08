---
id: 436
title: Package rebuilds in Modularity
date: 2017-10-02T13:43:08+00:00
author: adam
layout: post
guid: https://blog.samalik.com/?p=436
permalink: /package-rebuilds-in-modularity/
catchflames-sidebarlayout:
  - default
dsq_thread_id:
  - "6185570263"
categories:
  - Fedora
  - Modularity
---
People say that <a href="https://docs.pagure.org/modularity/" target="_blank" rel="noopener">Modularity</a> means rebuilding stuff over and over. Is that true? It might be for now. Important thing is that it&#8217;s not the goal, it&#8217;s just a first step. So why do we do that? It&#8217;s all about <a href="https://en.wikipedia.org/wiki/Application_programming_interface" target="_blank" rel="noopener">Application Programming Interface (API)</a>, <a href="https://en.wikipedia.org/wiki/Application_binary_interface" target="_blank" rel="noopener">Application Binary Interface (ABI)</a>, and automation.

Before we start, let&#8217;s refresh one of our goals. The goal is to make the whole process of building and releasing a distribution more efficient, and with the ability to ship more versions and features. That sounds like something we can&#8217;t do all at once.

The priority for us at the moment are humans. We want to automate as much of the packagers&#8217; workflow as we can. Even for the cost of computers doing significantly more work. We&#8217;ll make it easy for the computers later.

## The story of API and ABI compatibility

API compatibility is a compatibility on the source level. An example of this would be a package that builds for all releases (epel6, epel7, f26, f27, etc.) from the same source. While the source is the same, the binary for each release is different. The f27 binary might not work on epel6 etc.

ABI compatibility is a compatibility on the binary level. In this scenario, the same binary would run on all releases (epel6, epel7, f26, f27, etc.).

In the traditional Fedora release, we technically don&#8217;t expect any compatibility between releases. For every release of Fedora, we store the sources separately and we build them separately. That means the sources of all packages in the Fedora 26 release are stored in dist-git in an f26 branch, sources of Fedora 27 packages are in an f27 branch, etc.

So technically, there doesn&#8217;t need to be an API nor ABI compatibility between releases, because they are completely separated. However, in reality, many things are stable across multiple releases. If you are a packager, you might have a package that builds for all the releases from a single source just fine. That&#8217;s where Modularity helps.

Modularity introduces the concept of API compatibility between releases. Thanks to Arbitrary Branching, packagers can have a single branch of an application that is API compatible with multiple releases, and the build system will automatically build different binaries.

An important thing is that the build system will build the binaries automatically. We want to make the packagers&#8217; life as easy as possible, and that&#8217;s why there is a new service called Freshmaker that will automate two things: building binaries when the source changes, and rebuilding existing binaries in case of an ABI change. The most important word from this paragraph is &#8220;will&#8221;—I&#8217;m mostly talking about the future.

## The ultimate goal

The ideal state is that the build system knows exactly what needs to get built, and it reuses the binaries whenever possible.

In this ideal state, when a packager pushes an update to a certain package, the build system automatically rebuild the package to produce a binary for every release. If the package is ABI compatible with all the releases, it would build it once. If not, it would build it multiple times, but not more than needed.

The next step would be dependencies. If the package introduced an ABI change, it would automatically rebuild all its dependencies that wouldn&#8217;t work with this new ABI. In case the ABI is the same as before, no additional rebuilds would occur.

Implementing an automatic mechanism that would decide about when exactly to reuse a binary, and when to rebuild dependencies based on ABI changes is a rather complex task. While this is the ultimate goal, it&#8217;s not how it works these days.

## The current state

At the time of writing this post, the build system doesn&#8217;t know anything about ABI compatibility. But we have the same requirements—builds need to be automatic, and things need to work.

In theory, without any information about the ABI, the only way of being 100 % sure things will work is to rebuild everything. When a new version of a package is built, all its dependencies need to be rebuilt as well. Also all the dependencies of the dependencies needs to be rebuilt, etc. This extreme approach is not maintainable in the real world. We don&#8217;t want every update of glibc to trigger a rebuild of the whole distribution. So there needs to be a reasonable compromise.

The current implementation is a combination of limiting the rebuilds, trusting the ABI of module streams, and testing.

Modules are rebuilt automatically when one of their components change. Package rebuilds are done automatically only within the module, with a simple optimisation using build groups which I have described in my previous <a href="https://blog.samalik.com/2017/09/25/module-builds-how-do-they-work/" target="_blank" rel="noopener">Module builds – how do they work? </a>post.

However, rebuilding a module will not automatically trigger a rebuild of other modules having a dependency on it. This is where we need to trust the ABI stability of a module stream, together with testing the result after every build.

## Conclusion

So yes, there are some extra rebuilds, for now. Important is that this is not the goal. It is a first iteration of providing an automated build pipeline while making the the packagers&#8217; lives easier.