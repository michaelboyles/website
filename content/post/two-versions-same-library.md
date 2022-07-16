+++
author = "Michael Boyles"
title = "Using Two Versions of a Library in Java"
date = "2022-06-14"
description = "A tutorial on how to use two versions of the same library in a Java project"
tags = ["java", "maven"]
location = "London, England"
+++

Java and its build tools aren't designed to support using two different versions of the same library at runtime.
Generally, that's a conflict that's automatically resolved to one or the other, and that's exactly what we want. But
what if it's not? With a few tricks, it's possible to circumvent that behaviour.

<!--more-->

## The Problem

Suppose your project depends on two libraries which each depend on a third library, and that they drastically disagree
about which version they'd prefer.

```plaintext
MyApp
    \-> LibraryA v1.0
        \-> LibraryC v5.9
    \-> LibraryB v1.0
        \-> LibraryC v0.1
```

You need to choose *something* to put on the classpath at runtime. If you choose 5.9, there's a strong chance it will
break LibraryB. If you choose 0.1, there's a strong chance it will break LibraryA. You might even exhaustively
try every version in-between; there's no guarantee that any version will satisfy both.

The first thing you should be thinking about if you find yourself in this situation is whether you actually need both
LibraryA and LibraryB in the first place. If you can replace, rewrite or completely remove one of them then save yourself
the headache and do that.

Another solution would be to isolate one of either LibraryA or LibraryB behind a new microservice, though of course
this adds some overhead compared to doing everything in the same JVM.

If that's not feasible then welcome, my friend, to dependency hell. You can escape this place, but you'll have to make a
few deals with the devil. Even if you're lucky enough to never need the techniques described below, hopefully there's
some interesting things to learn about Java and Maven along the way.

As a slight simplification, I'm going to assume that our example application needs to directly depend upon two versions
of the same library, Apache Commons Lang 3.10 and 3.9, neither of which have any dependencies themselves. There's no
reason this approach wouldn't work for more complex trees too.

## The First Circle: Maven

If your project depends on two or more versions of the same library (specifically, the same group ID and
artifact ID) then all but one of them will be omitted. Maven will select the version which is highest in the dependency
tree. The theory is that something you depend on more closely is likely to be more important to your application
functioning correctly than something that you depend on very indirectly. This doesn't always prove true in practice, but
it's about the best guess it can make with the information it has.

Being able to omit conflicting dependencies is a key part of what allows Maven to work as well as it does. In a typical
project with perhaps dozens of direct dependencies and maybe hundreds of transitive ones, conflicting version
requirements are very common. Being forced to resolve them all manually would be painful and slow.

We need a way to subvert the standard resolution behaviour. In order to do that, we'll structure our app as a
multi-module project with three modules:

 - **app**, depends on sibling modules commonslang-3-9-0 and commonslang-3-10-0
 - **commonslang-3-9-0**, depends on the real commonslang 3.9
 - **commonslang-3-10-0**, depends on the real commonslang 3.10

We want to avoid introducing the 2 transitive dependencies of the real Commons Lang into the app module.
[Provided scope](https://maven.apache.org/guides/introduction/introduction-to-dependency-mechanism.html#Dependency_Scope)
prevents the transitive relationship, and was the first thing I tried, but it doesn't play well with the next step.
Runtime scope works best for our purposes.

```xml
<dependency>
   <groupId>org.apache.commons</groupId>
   <artifactId>commons-lang3</artifactId>
   <version>3.10</version><!-- and 3.9 in the other module -->
   <scope>runtime</scope>
</dependency>
```
*A section of the POM for the module commonslang-3-10-0*

## The Second Circle: Java

Even if we get both JARs onto our classpath, Java doesn't support loading two versions of the same class. The JVM has no
concept of "libraries" as units of code, and there's nothing preventing the same package from existing in multiple JARs.
The JVM deals with classes with fully qualified names (FQNs): `java.lang.String`, `java.util.List`, or `com.foo.MyClass`.

A basic application with a sequence of JARs on the classpath (`-cp a.jar,b.jar`) using the default [ClassLoader](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/lang/ClassLoader.html)
will search through each of those JARs in order until it finds a match for the FQN, then stop. If a class with
the same FQN exists in both `a.jar` and `b.jar` then the class in the latter will never be loaded.

This causes us an issue because it's extremely likely that at least some classes will exist with the same FQN in both
versions of our library; any overlapping classes can't both be loaded. The exact extent of the overlap will depend on
what's changed between versions. In our case, the version increment is small, so the overlap of classes is likely to be
large.

The solution is to repackage the libraries so that the classes have unique FQNs. The [Maven Shade Plugin](https://maven.apache.org/plugins/maven-shade-plugin/shade-mojo.html)
is a versatile plugin for repackaging dependencies which can do this for us.

One of the things the shade plugin allows you to do is to specify a package name and new destination to move them to.
This is more complex than it might first appear because it involves manipulating the bytecode for all those classes.
This results in a new "shaded" JAR.

For our application, we append a prefix that corresponds to the version of the library, in this case `three.ten`.

```xml
<groupId>org.apache.maven.plugins</groupId>
<artifactId>maven-shade-plugin</artifactId>
<version>3.3.0</version>
<configuration>
    <relocations>
        <relocation>
            <pattern>org.apache</pattern>
            <shadedPattern>three.ten.org.apache</shadedPattern>
        </relocation>
    </relocations>
</configuration>
```
*A section of the POM for the module commonslang-3-10-0*

The shade plugin is the reason we didn't use provided scope earlier, since provided-scope dependencies are not included
in the shaded JAR, while runtime-scope ones are.

The result is a JAR with the following structure:

```plaintext
     2399  03-23-2020 09:37   three/ten/org/apache/commons/lang3/AnnotationUtils$1.class
     6953  03-23-2020 09:37   three/ten/org/apache/commons/lang3/AnnotationUtils.class
        0  03-23-2020 09:37   three/ten/org/apache/commons/lang3/arch/
     1546  03-23-2020 09:37   three/ten/org/apache/commons/lang3/arch/Processor$Arch.class
     1333  03-23-2020 09:37   three/ten/org/apache/commons/lang3/arch/Processor$Type.class
     1619  03-23-2020 09:37   three/ten/org/apache/commons/lang3/arch/Processor.class
     ...
```
*Result of `unzip -l` on commonslang-3-10-0.jar*

We can now import and use classes in the `app` module such as:

```java
import three.ten.org.apache.commons.lang3.ClassUtils;
```

## The Third Circle: IDE

If you're developing in emacs or any other text editor then you're already done. Be sure to latch onto this rare, minor
victory and use it as the cornerstone of your argument as to why IDEs actually hinder productivity.

### IntelliJ

What we've done so far is perfectly valid and would build and run just fine, but IntelliJ doesn't detect what's
happening. That's a problem. It's not natively able to understand what the shade plugin is doing, and so you'll see imports for
`three.ten.org.apache.*` fail to resolve. There's [an open feature request](https://youtrack.jetbrains.com/issue/IDEA-93855)
but it's gone largely ignored for the past 10 years.

One workaround is to use the [Build Helper Plugin](https://www.mojohaus.org/build-helper-maven-plugin/usage.html) to
"attach" the shaded artifact. Attaching an artifact just involves registering a file, usually a JAR, with Maven.
It's something that certain plugins will usually just do automatically. The shade plugin can attach the shaded JAR too.
By default, it attaches the shaded JAR to replace the original output JAR.

For whatever reason, IntelliJ can grasp what's happened provided that you attach the shaded JAR with the helper plugin. I
don't think the way in which the 2 plugins attach the JAR is any different. I'd guess that IntelliJ just has a specific
handler for the helper plugin, and doesn't have one for the shade plugin.

You shouldn't have to change your build to cater to the deficiencies of an IDE, but I think it would be dogmatic to insist on
that. There's probably an alternative way to set things up in Project Structure, while leaving the build untouched, but
there's huge value in being able to check out a project and immediately run `mvn verify` without faffing around, and
checking IDE configuration into version control always ends up being annoying.

```xml
<groupId>org.codehaus.mojo</groupId>
<artifactId>build-helper-maven-plugin</artifactId>
<version>3.3.0</version>
<executions>
    <execution>
        <phase>package</phase>
        <goals>
            <goal>attach-artifact</goal>
        </goals>
        <configuration>
            <artifacts>
                <artifact>
                    <file>${project.build.directory}/${project.build.finalName}.jar</file>
                    <type>jar</type>
                    <classifier>shaded</classifier>
```
*A section of the POM for the module commonslang-3-10-0*

We use the [classifier](https://maven.apache.org/pom.html#dependencies:~:text=classifier) 'shaded' to attach the JAR in
addition to the unclassified JAR that's already attached, so we need to update the dependencies in our `app` module to
use those classifiers.

```xml
<dependency>
    <groupId>org.example</groupId>
    <artifactId>commonslang-3-12-0</artifactId>
    <version>${project.version}</version>
    <classifier>shaded</classifier>
</dependency>
<dependency>
    <groupId>org.example</groupId>
    <artifactId>commonslang-3-10-0</artifactId>
    <version>${project.version}</version>
    <classifier>shaded</classifier>
</dependency>
```
*A section of the POM for the module app*

### Eclipse

Eclipse might work out of the box. I mean, there's a first time for everything. Seriously though, I didn't try it.

## Talk is cheap. Show me the code!

A [full sample project](https://github.com/michaelboyles/version-shading) is available on GitHub.
