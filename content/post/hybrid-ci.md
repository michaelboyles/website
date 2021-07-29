+++
author = "Michael Boyles"
title = "Hybrid CI"
date = "2020-10-18"
description = "An approach to using two CI systems to get the best of both worlds"
tags = ["build automation", "ci"]
+++

The company I work for uses GitLab to host our Git repositories. Along with [GitHub more recently](https://techcrunch.com/2019/08/08/github-actions-is-now-a-ci-cd-service/), it provides a way to build your projects directly within the service without relying on an external build server like Jenkins or Travis. The general reaction here seems positive, and most people seem happy to rely on it for their only automated build now. I can see the benefits but I’m hesitant to fully commit. Hybrid CI is an approach I came up with so I could have my cake and eat it too.

<!--more-->

## Reasons to use GitLab

**Rich integrations**. By far the most compelling reason to build within your repository manager is that you get rich feature interactions between your code and your build, because you have just one system which is aware of both functions. You can trigger builds automatically when branches change without worrying about setting up webhooks. Being able to see the build status for each individual branch as you navigate the repository in your browser can be very helpful. Preventing a merge if the source branch fails the build is a great way to enforce quality.

A similar set of integrations can be achieved on GitHub while still using an external build by using the [Checks API](https://docs.github.com/en/free-pro-team@latest/rest/reference/checks) - effectively an API for build status callbacks - though it’s still something you need to manually configure. GitLab [doesn’t have this feature](https://gitlab.com/gitlab-org/gitlab-foss/-/issues/46145).

**Simple configuration**. GitLab provides a series of [templates](https://gitlab.com/gitlab-org/gitlab-foss/tree/master/lib/gitlab/ci/templates) to get a build up and running quickly and those are going to be sufficient for most build processes, unless yours is particularly idiosyncratic. In my experience, Jenkins is harder to get up and running exactly how you like it, especially the first time.

## Reasons to use Jenkins

**Huge plugin library**. Jenkins already has [more than 1500 plugins](https://plugins.jenkins.io/) to accomplish pretty much any task you can imagine.

Companies like GitLab [will try to convince you that this is a bad thing](https://about.gitlab.com/blog/2019/09/27/plugin-instability/) but of course they’re biased. It’s in their best interest if you’re fully invested in their platform, to keep you locked in, and to keep you paying.

**Extensible**. Can’t find a plugin which matches your needs? You can easily write your own because Jenkins was designed with extensibility in mind. There’s no need to sit around, waiting to see if your feature request will be implemented. Let’s face it, it probably won’t be.

In terms of delivering features, it’s simply not possible for their limited number of developers to compete with the manpower of an established community.

## Why choose?

It might be because I’m a *bloody millennial* but I couldn’t help wondering why I should have to choose between the two. So I don’t. In all my projects now, we run two different builds: one directly within Gitlab and one on Jenkins. The GitLab build compiles the code and does some basic checks for code style etc. The Jenkins pipeline is more comprehensive. It creates reports for code coverage, plots graphs for test failures, runs static analysis, deploys artifacts, and more.

The extra configuration is minimal since the GitLab version almost always follows an identical template: run `mvn verify`.

There are some side benefits to this approach too. We run all our builds on Windows and Linux where before we may not have bothered; most of our projects are Java and so aren’t terribly platform-specific but it’s still sometimes handy. GitLab also makes it very difficult to manage Maven repository credentials, so by deploying from another system we avoid that headache.

