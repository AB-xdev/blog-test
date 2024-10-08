---
title: |
  Discovering the perfect Java Supply Chain Attack vector and how it got fixed
author: AB
categories:
  - Blog
tags:
  - java
  - compiler
  - annotations
  - annotation-processor
---

You may have heard about the [changes to annotation processing that have been done in Java 23](https://www.oracle.com/java/technologies/javase/23-relnote-issues.html#JDK-8321314).<br/>
This post will cover the underlying topic, how I learned about it, and how it got fixed.

## Background

To understand annotation processing and why it is even there, we first need to hop into our DeLorean and go back to the year ~~1955~~ 2005.

So back then there were no build tools like Maven or Gradle, which we have nowadays. Java was still developed by Sun Microsystems and the newest version was 5.

Java annotations were introduced with Java 5 but an API for processing them at build-time did not exist, so [JSR 269](https://www.jcp.org/en/jsr/detail?id=269) was born.

The general idea seems to have been to allow the generation of (boilerplate) code and similar operations. Examples that use the concept nowadays are probably [Lombok](https://projectlombok.org/) or [Dagger](https://github.com/google/dagger) (for the Android world) but these are usually run with dedicated plugins.<br/> I found [an old video of a presentation](https://www.youtube.com/watch?v=0hN6XJ69xn4) about this topic - you may want to check it out for additional background information.

JSR 269 was ultimately implemented and shipped in 2006 with Java 6.
Most details of it were likely ignored over the years as better build tools such as Maven or Gradle gradually gained popularity.

## How the ~~dog~~ compiler ate my records...

Back to the present - well nearly: Let's stop on a day in late 2022.

So back then we updated one of our bigger Maven-based projects to Java 17, we converted a few classes to the new  ``records``. Everything compiled without issues so we shipped the project.

A few days later I was again adding a ``record``, and tried to compile it, when all of a sudden the compiler crashed with the following message:
```
[ERROR] org.apache.maven.lifecycle.LifecycleExecutionException: Failed to execute goal org.apache.maven.plugins:maven-compiler-plugin:3.10.1:compile (default-compile) on project X: Compilation failure
...
Caused by: org.apache.maven.plugin.compiler.CompilationFailureException: Compilation failure
C:\...\ClassUsingARecord.java: Internal compiler error: java.lang.Exception: javax.lang.model.element.UnknownElementException: Unknown element: \"RecordClass\" 
    at org.eclipse.jdt.internal.compiler.apt.dispatch.RoundDispatcher.handleProcessor(RoundDispatcher.java:172)
    at org.apache.maven.plugin.compiler.AbstractCompilerMojo.execute (AbstractCompilerMojo.java:1310)
    at org.apache.maven.plugin.compiler.CompilerMojo.execute (CompilerMojo.java:198)
    ...
    at org.codehaus.classworlds.Launcher.main (Launcher.java:47)
```

The error looked pretty weird (I had never seen a ``UnknownElementException`` before) and after a bit of debugging I came to the following conclusions:
* Yes I was indeed using Java 17 - which supports ``records`` - and not some other version
* Coworkers had the same problem when checking out the code
* The problem was not present on any other Java 17 projects
* The problem only occurred in a few Maven modules, which didn't seem to like the  ``record`` keyword

After running out of ideas I looked again at the stacktrace and noticed that the exception occurred on ``handleProcessor`` which - after some research - turned out to be for an annotation processor.

So let's simply disable all annotation processors and this fixes everything?

Great idea... but there was just one problem:<br/>**There were no annotation processors** - at least none were defined in the configuration of ``maven-compiler-plugin``.

### So where did that annotation processor come from and why was it executed at build time?

As it turns out: In one of the upstream Maven modules - that was inside of all the modules that had the problem - ``hibernate-validator-annotation-processor`` was defined as a dependency.
This **outdated library** was unused for years and had likely been forgotten during a cleanup.
The library **contains an annotation processor**: ``ConstraintValidationProcessor`` and **that processor [was unable to handle Java 17 code](https://hibernate.atlassian.net/browse/HV-1863)**.

The problem was obviously fixed by removing the dependency but that annotation processor are somehow automatically loaded and executed during build time without any notice confused me a bit...

### Down the rabbit hole

So the next day I did some follow-up research and discovered basically everything that you already read before in the "Background" section above.

Well I didn't tell you everything because deep in the JavaDocs I found this:

> **Unless annotation processing is disabled with the `-proc:none` option, the compiler searches for any annotation processors that are available.** The search path can be specified with the `-processorpath` option. If no path is specified, then the user class path is used. Processors are located by means of service provider-configuration files named `META-INF/services/javax.annotation.processing` ... <sup>[Java Docs](https://docs.oracle.com/en/java/javase/17/docs/specs/man/javac.html#annotation-processing)</sup>

Now the above behavior made sense (there was a [Java Service Loader](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/util/ServiceLoader.html) file inside the dependency in the form of ``META-INF/services/javax.annotation.processing.Processor``) but that description really hit me as no other programming language/compiler that I knew __executes code__ that it should compile... by default.

## The perfect vector

Also at the same time, a lot of alarm bells started ringing because a while ago I read about some supply chain attacks in NPM and this was just the manifestation of the same attack vector:

Consider the following attack scenario:
* Your project is using library ``L``
* You did enable automatic dependency updates in the form of e.g. [Renovate](https://github.com/renovatebot/renovate) or [Dependabot](https://github.com/dependabot)
* The created PRs from the updates are compiled by your CI
* Now library ``L`` suffers a supply chain attack and a malicious annotation processor + Service Loading file is inserted into the library
* A new malicious version of library ``L`` is released
* The automatic dependency update tool picks up the new version and creates a PR
* Your CI - that just compiles the PR - may get compromised **during compilation**
  * This can also be abused to steal secrets present during the build e.g. access keys for registries
  * If the build is not sand-boxed a lot more damage can be done
* A developer checking out the update may be compromised as soon as starting the IDE
  * This can likely not be detected by a antivirus that only uses (file) signatures for detection since it's exploited using the compiler

The following points make this quite dangerous:
* Stealth
  * There is no indication from the compiler that it now executes a annotation processor (before Java 21)
    * Most developers don't know about this annotation processor auto-discovery behavior (none that I asked at my company which has years of Java experience knew of it)
  * As it runs at build time there is likely no trace visible in the final output
  * A malicious processor can hide perfectly between legitimate ones
  * Without manual review of each dependency (JAR) it's extremely unlikely that anyone or anything will ever notice it - especially in big projects with hundreds of dependencies
* Execution is triggered nearly instantly
  * The processor is triggered on compilation - which happens in IDEs and CIs all the time

## We didn't start the fire... - But how do we put it out?

So at first I had a look what exactly is effected by this.

I created a demo "malicious" processor that tries to open a URL and then terminates the compile process using ``System.exit`` - you may check it out [here](https://github.com/AB-xdev/java-annotation-processing-code-execution) - and tested it against various things:

* Compilers
  * Java's default compiler ``javac``
  * Eclipse compiler ``ecj``
* Build tools that use these compilers
  * Maven's ``maven-compiler-plugin``
  * Gradle is most likely not affected as it usually uses a [Java Toolchain](https://docs.gradle.org/current/userguide/toolchains.html) and annotation processors must be declared explicitly
* IDEs
  * IntelliJ IDEA results vary based on the used build tool.<br/>Executing a Maven build with the demo causes the build process.
  * AFAIK Eclipse doesn't run annotation processing by default, however if enabled:<br/>The build-process isn't sand-boxed in any way - this causes an IDE crash

### Short term fixes and workarounds

Well luckily there is a flag that disables annotation processing during compile:
[``-proc:none``](https://docs.oracle.com/en/java/javase/17/docs/specs/man/javac.html#option-proc)

Simply add it to your compiler arguments and you should be fine.

Example for Maven:
```xml
<plugin>
	<groupId>org.apache.maven.plugins</groupId>
	<artifactId>maven-compiler-plugin</artifactId>
	<configuration>
		<compilerArgs>
			<arg>-proc:none</arg>
		</compilerArgs>
	</configuration>
</plugin>
```

If you need to run an annotation processor anyway:<br/>
Explicitly run the annotation processor in an additional compile step.

Example for Maven using the ``maven-processor-plugin``:
```xml
<plugin>
    <groupId>org.bsc.maven</groupId>
    <artifactId>maven-processor-plugin</artifactId>
    <executions>
        <execution>
            <id>process</id>
            <goals>
                <goal>process</goal>
            </goals>
            <phase>generate-sources</phase>
            <configuration>
                <processors>
                    <processor>org.hibernate.processor.HibernateProcessor</processor>
                </processors>
            </configuration>
        </execution>
    </executions>
</plugin>
```

For IDEs you can usually check the settings if annotation processing is enabled or not.<br/>
IntelliJ IDEA also automatically imports the configuration from Maven projects.

### Getting this fixed - globally

Parallel to applying fixes to existing (company) projects I also filled a [JDK bug report](https://bugs.openjdk.org/browse/JDK-8306819) and after some time it was marked as [received](/assets/blog/java-annotation-processing-design-flaw/JDK-Bug-Report-Received.png).

The following things happened since then:

| Issue | Description | Java Version |
| --- | --- | --- |
| [JDK-8310061](https://bugs.openjdk.org/browse/JDK-8310061) | If implicit annotation processing is detected a note is printed by the compiler | 21+ |
| [JDK-8308245](https://bugs.openjdk.org/browse/JDK-8308245) | For parity reasons the ``-proc:full``-flag was backported to Java LTS versions | 8, 11, 17 |
| [JDK-8321319](https://bugs.openjdk.org/browse/JDK-8321319) | Annotation processing disabled by default | 23+ |

## Acknowledgements
A special thanks to
* Anyone from OpenJDK who contributed to this topic
  * Especially to [Joe Darcy](https://github.com/jddarcy) who did most of the implementations (as far as I could tell)
* [XDEV Software](https://xdev.software) which made it possible to write this post
  * [JR](https://github.com/JohannesRabauer) for helping out with the post
