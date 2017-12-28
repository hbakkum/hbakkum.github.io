---
layout: post
title: "Creating a Simple Dropwizard Microservice in Kubernetes"
comments: true
description: ""
keywords: "kubernetes dropwizard microservices"
author: Hayden Bakkum
visible: 0
---

In this post I'll describe how to create a simple dropwizard based microservice and deploy this into a kubernetes cluster. For the purposes
of this demo, I'll be using [Google's Kubernetes Engine](https://cloud.google.com/kubernetes-engine/) as the kubernetes cluster provider.

In order to get a dropwizard service deployed to kuberenetes, a few basic building blocks are required:
* a docker image that starts a dropwizard service
* a kubernetes configuration file that references said image and describes how it should be deployed to the cluster (see [kubernetes deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/))
* a build pipeline to trigger the image build and then manage the deployment to kubernetes  

The service used as an example will expose a REST endpoint for returning a list of movies and will be aptly named the **movies-service**. 
Full source code for the movies-service can be found here (TODO: link).

The project layout of the service can be seen in the image below, with a brief description against the various files which we'll now
explore in more detail.
![Movies Service Project Layout]({{ site.url }}/assets/images/movies-service-project-layout.png)

To begin with, we'll look at the pom file in more detail. As previously mentioned, one of the core building blocks required to get a dropwizard service
running in kubernetes, is a docker image that starts a dropwizard service. The pom file thus includes the configuration required to first build a jar containing
the dropwizard service and then build a docker image that includes this jar (and its dependencies).

Building the dropwizard jar is pretty standard and the following pom snippet shows how this is done:
{% highlight xml %}
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-jar-plugin</artifactId>
    <version>3.0.2</version>
    <configuration>
        <archive>
            <manifest>
                <addClasspath>true</addClasspath>
                <classpathPrefix>lib/</classpathPrefix>
                <mainClass>com.kiwitype.blog.k8s.services.movies.MoviesService</mainClass>
                <addDefaultImplementationEntries>true</addDefaultImplementationEntries>
            </manifest>
        </archive>
    </configuration>
</plugin>
{% endhighlight %} 

One difference between the [recommended way of building a dropwizard jar](http://www.dropwizard.io/1.2.2/docs/getting-started.html#building-fat-jars) 
is that we do not build a fat jar. In the dropwizard documentation, the argument made for doing this is that building a fat jar makes it easier to promote the 
same artifact between repositories. This is true (and a great practise as opposed to say rebuilding for different environments), 
however in our case we will not be promoting jars between repositories but rather docker images, so we can easily include all dependencies outside of the jar 
but within the same image. A benefit of this is that we can make the build more efficient by avoiding the overhead of executing the [Shade Plugin](https://maven.apache.org/plugins/maven-shade-plugin/), 
as well as taking full advantage of docker layer caching by including all dependencies in their own layer. Doing this ensures that the final 
image layer (the layer that includes the dropwizard jar) is as small as possible. 

You should also be able to see that we have added a classpath entry to the dropwizard jar that we will use as the location for its dependencies. Looking
further ahead in the pom the maven dependency plugin is used to copy these dependencies into the target directory so they can be copied into the
docker image:
{% highlight xml %}
<plugin>
    <artifactId>maven-dependency-plugin</artifactId>
    <executions>
        <execution>
            <phase>initialize</phase>
            <goals>
                <goal>copy-dependencies</goal>
            </goals>
            <configuration>
                <overWriteReleases>false</overWriteReleases>
                <includeScope>runtime</includeScope>
                <outputDirectory>${project.build.directory}/lib</outputDirectory>
            </configuration>
        </execution>
    </executions>
</plugin>

</project>
{% endhighlight %} 

Finally we build the docker image. To do this the [Spotify Dockerfile Maven Plugin](https://github.com/spotify/dockerfile-maven) is used. This plugin looks for 
a Dockerfile in the projects root directory (by default) and uses this to run a docker build. 
{% highlight xml %}
<plugin>
    <groupId>com.spotify</groupId>
    <artifactId>dockerfile-maven-plugin</artifactId>
    <version>1.3.6</version>
    <executions>
        <execution>
            <id>default</id>
            <goals>
                <goal>build</goal>
                <goal>push</goal>
            </goals>
        </execution>
    </executions>
    <configuration>
        <repository>us.gcr.io/${gcloud.project}/${project.artifactId}</repository>
        <tag>${project.version}</tag>
        <buildArgs>
            <JAR_FILE>${project.build.finalName}.jar</JAR_FILE>
        </buildArgs>
    </configuration>
</plugin>
{% endhighlight %}

The goals **build** and **push** are declared and these bind to the maven **package** and **deploy** phases respectively. Thus a **mvn clean package** can be used to produce
the docker image and a **mvn clean deploy** will produce the image as well as push it to a docker repository. For this example, the docker repository being
used is [Google's Container Registry](https://cloud.google.com/container-registry/). 

By default the plugin will look for a **Dockerfile** in the same directory as the pom file. The contents of this Dockerfile are as follows:
{% highlight docker %}
FROM openjdk:8-jre

# declare that the container listens on these ports
EXPOSE 8080
EXPOSE 8081

# standard command for starting a dropwizard service
ENTRYPOINT ["/usr/bin/java", "-jar", "/usr/share/movies-service/movies-service.jar", "server", "/usr/share/movies-service/config.yml"]

# add in project dependencies
ADD target/lib /usr/share/movies-service/lib

# add dropwizard config file - the server is configured to listen on ports 8080 (application port) and 8081 (admin port)
ADD target/config/dw-config.yml /usr/share/movies-service/config.yml

# add built dropwizard jar file - the JAR_FILE argument is configured in the dockerfile maven plugin 
ARG JAR_FILE
ADD target/${JAR_FILE} /usr/share/movies-service/movies-service.jar
{% endhighlight %}

Thus running a **mvn clean deploy** will build our dropwizard image and push it to container registry, ready for deployment to kubernetes.

And one final note on the build - the maven deploy plugin has been configured to skip deployment of the dropwizard jar to a maven repository as
really the build artifact is the docker image which has already been pushed to the container registry.

The next thing we need to do is produce the kubernetes configuration 

And finally we need to define a build pipeline for coordinating the build and deployment to kubernetes. For this, 
I'll be using Jenkins. The following Jenkinsfile:
{% highlight groovy %}
FROM openjdk:8-jre

# declare that the container listens on these ports
EXPOSE 8080
EXPOSE 8081

# standard command for starting a dropwizard service
ENTRYPOINT ["/usr/bin/java", "-jar", "/usr/share/movies-service/movies-service.jar", "server", "/usr/share/movies-service/config.yml"]

# add in project dependencies
ADD target/lib /usr/share/movies-service/lib

# add dropwizard config file - the server is configured to listen on ports 8080 (application port) and 8081 (admin port)
ADD target/config/dw-config.yml /usr/share/movies-service/config.yml

# add built dropwizard jar file - the JAR_FILE argument is configured in the dockerfile maven plugin 
ARG JAR_FILE
ADD target/${JAR_FILE} /usr/share/movies-service/movies-service.jar
{% endhighlight %}

 
 
 
 
Before diving into the service in detail, I'lll give a brief overview of the project files. 


The build pipeline
- Build a docker image containing the dropwizard jar                           
- Upload image to docker registry (in this case google container registry)
- Deploy

The tools used to do this include:
- [dropwizard](http://www.dropwizard.io)
- [apache maven](https://maven.apache.org/) (build tool)
- [dockerfile maven plugin](https://github.com/spotify/dockerfile-maven) (maven plugin to build docker image)
- [jenkins](https://jenkins-ci.org/)
- [google container registry](https://cloud.google.com/container-registry/) (as the docker image registry)
- [google kubernetes engine](https://cloud.google.com/kubernetes-engine/) (as the kubernetes cluster provider) 

// why avoid building a fat jar
// disable jar deploy

// jenkins stash

// mention about image promotion

// health checks

// making request to service