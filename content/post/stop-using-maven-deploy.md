+++
author = "Michael Boyles"
title = "Stop Using Maven Deploy"
date = "2020-08-24"
tags = ["build automation", "ci"]
+++


[`mvn deploy`](https://maven.apache.org/plugins/maven-deploy-plugin/) is the final phase of [Maven's build lifecycle](https://maven.apache.org/guides/introduction/introduction-to-the-lifecycle.html). It’s responsible for uploading the artifacts produced by the previous phases of the build to an artifact repository like [Nexus](https://www.sonatype.com/product-nexus-repository) where they can be stored and accessed by other dependent projects, or used as part of an automated deployment.

Maven is a build tool. Its responsibility should be to build the application. Tacking on an artifact upload step at the end seems convenient and harmless enough, but in practice I’ve found that I prefer to avoid using it.

The major problem as I see it is that there is that it’s difficult to differentiate between a developer’s machine and the [build server](https://en.wikipedia.org/wiki/List_of_build_automation_software#Continuous_integration). There should be no reason to upload artifacts from a development machine, and you should try your best to make doing so absolutely impossible. If you don't then it becomes possible to circumvent any quality checks that are normally enforced by the build server. It even allows you to push a build containing code that was never committed to source control. If you end up in that situation, good luck decompiling your JARs to try to work out what changes were included.

One solution is to enforce authentication on your artifact repository, which of course you should be doing anyway. If you have a username and password which only the build server knows, it would be impossible to upload artifacts from anywhere else.

The problem I’ve found with this is that Maven does not make it easy to keep the password secret. I’ve often seen my colleagues sharing passwords for artifact repos in plaintext or, worse, checking them into source control.

Maven does provide [a way to encrypt passwords](https://maven.apache.org/guides/mini/guide-encryption.html) but it relies on keeping configuration files on the local filesystem, and keeping those files secret; you could try changing the unix file permissions for those files but I find that unreliable. It can be a hassle to update the passwords if the files exist on multiple runners. They could also be easily missed if you spin up a new build server and have nothing to document that they’re required.

The solution which has worked for me is to move the artifact upload step to a separate stage of the build pipeline, after Maven has completed. I don't use any `<distributionManagement>`. I use Jenkins and the [Nexus Artifact Uploader plugin](https://plugins.jenkins.io/nexus-artifact-uploader/). The only real downside is having to manually enumerate the artifacts. In projects with a few modules it can get a bit tedious, but I’ve sometimes appreciated the fine-grained control of being able to exclude a module or two by simply omitting them from the list.

The section of my [build pipeline](https://www.jenkins.io/pipeline/getting-started-pipelines/) responsible for deployment will often look something like this:


```groovy
nexusArtifactUploader {
    nexusVersion('nexus3')
    protocol('https')
    nexusUrl('our-nexus-host')
    groupId('com.mycorp')
    version('1.2.3')
    repository('my-repo-snapshots')
    credentialsId('44620c50-1589-4617-a677-7563985e46e1')
    artifact {
        artifactId('my-app')
        type('jar')
        file('target/my-app.jar')
    }
    artifact {
        artifactId('my-app')
        type('pom')
        file('pom.xml')
    }
}
```

The advantage here is that `credentialsId` is a *reference* to a set of credentials but the mechanism by which those credentials are obtained is left up to the build server. I could store the password directly in Jenkins, or in a separate secrets manager like [HashiCorp Vault](https://www.vaultproject.io/) (which is what we do). Either way, the credentials cannot be easily retrieved by a user, are easy to update, and their usage is fully documented in the pipeline definition.

There are other advantages too. Splitting the artifact upload from the build means you’ll get a more granular view of where a pipeline has failed. If Maven and the upload are separate steps in the pipeline, then I can see quickly which step has failed and I know immediately where to start looking. If Maven were responsible for both, both failures would be conflated into a single step, and only the build output would reveal what went wrong.

