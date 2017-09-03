---
layout: post
title: "Maven Versioning in a Continuous Delivery Pipeline"
comments: true
description: ""
keywords: "Continuous Delivery Jenkins Maven"
author: Hayden Bakkum
---

In this post I'll discuss some recent experience I've had setting up a continuous delivery pipeline using Maven and Jenkins.
In particular, I'll focus on some of the challenges encountered with using a dynamic maven version throughout a projects pom hierarchy.

One of the features of a continuous delivery pipeline is that each commit to source control is a potential production deployment. The first step in the pipeline is
typically a software build that is triggered upon commit to source control (e.g. via a CI server) and this produces one or more deployable packages.
These packages are considered to be release candidates which the pipeline then carries through a series of validating steps that attempt to reject the
packages as being suitable for deployment to production. The final phase of the pipeline, should the packages pass successfully through all prior steps,
is deployment to production (that's not to say the entire process completes without any manual intervention, there may be, for example, manual testing or
approval steps within the pipeline).

When using Maven as the build tool in a continuous delivery pipeline, I'd like to have the following behaviour:

 - Every build is a potential release, so I want to use a maven release version (i.e. not a snapshot version) for each build, and
 - Each build gets a release version assigned that has been incremented from the previous builds release version

I'd first like to rule out the [maven release plugin](http://maven.apache.org/maven-release/maven-release-plugin/) as satisfying these requirements.
Although we could trigger the maven release plugin to auto increment a projects version during the build, the plugin will commit the version changes
in the pom files back to source control, which will then trigger the build again, resulting in a infinite loop of builds.

An alternative approach, is to use a property placeholder in the projects parent pom version field which would be used to inject a
build number from a CI server (in my case Jenkins). This approach can be used to produce an incrementing release version for every commit to the
repository without the need to modify the pom and commit this back into source control. For builds that occur outside of Jenkins
(e.g. on a developers local machine) we can simply replace the build number with the string "**local**".

The following parent pom demonstrates how this can be done:

{% highlight xml %}
<project>

    <groupId>com.hbakkum</groupId>
    <artifactId>app-parent</artifactId>
    <version>1.${build.number}</version>
    <packaging>pom</packaging>

    <name>My App</name>

    <profiles>
        <profile>
            <id>jenkins-build</id>
            <activation>
                <property>
                    <name>env.BUILD_NUMBER</name>
                </property>
            </activation>
            <properties>
                <build.number>${env.BUILD_NUMBER}</build.number>
            </properties>
        </profile>
    </profiles>

    <properties>
        <build.number>local</build.number>
    </properties>

    <modules>
        <module>app-core</module>
        <module>app-web</module>
        <module>app-spec-tests</module>
    </modules>

    ...

</project>
{% endhighlight %}

In summary:
- as mentioned, by default (i.e. when not running the build from jenkins) the build number will be set to "**local**"
- a profile gets activated (via the presence of a environment variable provided by jenkins: **BUILD_NUMBER**) when the build was triggered by jenkins. This
build number gets injected into the projects parent pom version field
- the parent pom version gets inherited by all child modules of the project
- changing the projects major version (in the example it is "1") is a human decision and can be done by simply updating the parent pom and committing the change.

Some readers may recognise that maven explicitly warns against using properties in a poms version field and indeed when a maven build is run the following
warnings appear:

{% highlight console %}
[WARNING]
[WARNING] Some problems were encountered while building the effective model for com.hbakkum:app-parent:pom:1.local
[WARNING] 'version' contains an expression but should be a constant. @ com.hbakkum:app-parent:1.${build.number}, blog/pom.xml, line 11, column 14
[WARNING]
[WARNING] Some problems were encountered while building the effective model for com.hbakkum:app-parent:pom:1.local
[WARNING] 'version' contains an expression but should be a constant. @ com.hbakkum:app-parent:1.${build.number}, blog/pom.xml, line 11, column 14
[WARNING]
[WARNING] It is highly recommended to fix these problems because they threaten the stability of your build.
[WARNING]
[WARNING] For this reason, future Maven versions might no longer support building such malformed projects.
[WARNING]
{% endhighlight %}

It does surprise me that a tool as widely used as maven still makes integration with a continuous delivery pipeline so difficult.
A number of years ago [MNG-5576](https://issues.apache.org/jira/browse/MNG-5576) was resolved which went a little way towards making this easier.
This issue loosened this restriction by not warning against the use of a few specific properties in the version field.

The allowed properties are:
- **${revision}**
- **${sha1}**
- **${changelist}**

However, one property I feel is missing from this list is **${build.number}** as I fail to see the difference between allowing an
incrementing sequence supplied by subversion versus an incrementing sequence supplied via Jenkin.
And although I am using git, I certainly don't want to see a sha1 in my version string!
So I choose to simply ignore this warning (and anyway, if I really wanted to,
I _could_ have just shoe-horned the build number into the revision property to prevent the warnings).

However, another much more serious problem looms whether you use revision, sha1, changelist or a "disallowed" property in the poms version.
This occurs when a project is multi moduled with child modules that reference the parent pom using a property placeholder in the version.

As an example, suppose we had the following parent pom:

{% highlight xml %}
<project>

    <groupId>com.hbakkum</groupId>
    <artifactId>app-parent</artifactId>
    <version>1.${build.number}</version>
    <packaging>pom</packaging>

    <modules>
        <module>app-core</module>
        <module>app-spec-tests</module>
    </modules>

    ...

</project>
{% endhighlight %}

And these two child modules, with a dependency from app-spec-tests to app-core:
{% highlight xml %}
<project>

  <parent>
      <groupId>com.hbakkum</groupId>
      <artifactId>app-parent</artifactId>
      <version>1.${build.number}</version>
  </parent>

  <artifactId>app-core</artifactId>

  ...

</project>
{% endhighlight %}

{% highlight xml %}
<project>

    <parent>
        <groupId>com.hbakkum</groupId>
        <artifactId>app-parent</artifactId>
        <version>1.${build.number}</version>
    </parent>

    <artifactId>app-spec-tests</artifactId>

    <dependencies>
        <dependency>
            <groupId>com.hbakkum</groupId>
            <artifactId>app-core</artifactId>
            <version>${projection.version}</version>
        </dependency>
    </dependencies>

    ...

</project>
{% endhighlight %}

Things work fine if we say run a '**mvn clean install**' from the parent pom level.
However, if we wanted to just build from the app-spec-tests module only, the build fails with the following message:

{% highlight console %}
[INFO] Building App Spec Tests 1.local
[INFO] ------------------------------------------------------------------------
Downloading: https://repo.maven.apache.org/maven2/com/hbakkum/app-parent/1.$%7Bbuilder.number%7D/app-parent-1.$%7builder.number%7D.pom
[INFO] ------------------------------------------------------------------------
[INFO] BUILD FAILURE
[INFO] ------------------------------------------------------------------------
[INFO] Total time: 1.839 s
[INFO] Finished at: 2017-05-18T20:38:24+12:00
[INFO] Final Memory: 9M/192M
[INFO] ------------------------------------------------------------------------
[ERROR] Failed to execute goal on project service: Could not resolve dependencies for project com.hbakkum:app-spec-tests:jar:1.local:
        Failed to collect dependencies at com.hbakkum:app-core:jar:1.local: Failed to read artifact descriptor for com.hbakkum:app-core:jar:1.local:
        Could not find artifact com.hbakkum:app-parent:pom:1.${build.number} in central (https://repo.maven.apache.org/maven2) -> [Help 1]
{% endhighlight %}

Notice how when maven is trying to resolve the parent pom of the **com.hbakkum:app-core:jar:1.local** dependency, that it does not use
a path with the resolved version for the pom, rather it uses the unresolved property placeholder version:

**/com/hbakkum/app-parent/1.$%7Bbuilder.number%7D/app-parent-1.$%7builder.number%7D.pom**

The parent pom never gets installed (rightly so) under this path and thus the dependency resolution fails. This will also occur when building an external maven project that depends on any of these
modules too.

### The Maven Flatten Plugin
Fortunately, there is a maven plugin that can resolve this issue. It was a little difficult to find, but after reading the discussions
 on [MNG-5576](https://issues.apache.org/jira/browse/MNG-5576) I found the [Maven Flatten Plugin](http://www.mojohaus.org/flatten-maven-plugin/).

The Maven Flatten Plugin can be used to generate an alternative version of your projects pom file that maven will then use to install and deploy.
Some of the features of this generated pom include merging in the parent pom information (and then removing the parent pom reference) and resolving all
property placeholders, thus addressing the issue described above.

This plugin does solve the problem, but there is an issue you need to look out for (and one that became a bit of a time sink to debug).
You'll notice that the plugin writes the generated pom file into the modules root directory (as opposed to the target directory) and the plugin has an explicit goal to clean this file up as part of the
maven clean phase.

I found this a tad annoying and questioned why the default location is not within the modules target directory.
There is plugin configuration to control the output directory of the generated pom though, so I was tempted to change this to be inside the target directory.
This should come at a huge warning though (see this [github issue](https://github.com/mojohaus/flatten-maven-plugin/issues/50) I created for a summary). Upon making this
change it was noticed that my integration tests were no longer run during the maven build.

After a lot of debugging and reading through the flatten plugin source code, I uncovered the cause of this issue. The flatten plugin uses [this method](http://maven.apache.org/ref/3.1.1/apidocs/org/apache/maven/project/MavenProject.html#setFile(java.io.File))
to change the location of the projects pom file. However, this has the additional side effect of changing the **${basedir}** of your maven modules to be
the directory containing the pom file. For me, this was causing an issue with adding new test sources (via the [Build Helper Maven Plugin](http://www.mojohaus.org/build-helper-maven-plugin/))
as I was using relative paths to the additional test source directories and the build helper plugin was resolving these paths against **${basedir}** which had
now changed location. What made this extra hard to track down was the fact that the basedir was only changed *after* the flatten plugin had run.

So one solution to this is really to just live with the default output location for the generated pom (which you'll probably want to add to your scm ignore list) as there are too many
risks associated with your basedir changing at some point during your build. However, in the end I decided to write my own plugin, the
[Resolve Parent Version Plugin](https://github.com/hbakkum/resolve-parent-version-maven-plugin).

The plugin makes use of a new api method in maven **3.2.5** that allows you to override the pom file location without altering your basedir
(see [MavenProject#setPomFile(java.io.File)](http://maven.apache.org/ref/3.2.5/apidocs/org/apache/maven/project/MavenProject.html#setPomFile(java.io.File))).
This allows the plugin to, by default, generate the pom file in the /target directory which is likely already scm ignored and automatically cleaned but does have the restriction
of requiring at least maven **3.2.5**. I've also restricted this plugin to modify the pom file in a very specific way, that is, all it will do is simply resolve the version of the parent
module to ensure there are no property placeholders present. This targets only the issue discussed and leaves the rest of the pom unmodified as I did not require any of the other
changes being made by the flatten plugin.








