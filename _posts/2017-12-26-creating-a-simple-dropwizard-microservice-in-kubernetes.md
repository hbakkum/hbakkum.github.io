---
layout: post
title: "Creating a Simple Dropwizard Microservice in Kubernetes"
comments: true
description: ""
keywords: "kubernetes dropwizard microservices"
author: Hayden Bakkum
visible: 0
---

In this post I'll describe how to create a simple dropwizard based microservice and deploy this into a kubernetes cluster.

The basic building blocks required to get a dropwizard service running in kubernetes are:
    - a docker image that starts a dropwizard service and
    - a kubernetes configuration file that references said image and describes how it should be deployed to the cluster (see kubernetes deployment)  

The service used as an example will expose a REST endpoint for returning a list of movies and will be aptly named the "movies-service". 
Full source code for the movies-service can be found here (TODO: link).

The project layout of the service can be seen in the image below, with a brief description against the various files which we'll now
explore in more detail.
![Movies Service Project Layout]({{ site.url }}/assets/images/movies-service-project-layout.png)

To begin with, we'll look at the pom file in more detail. As previously mentioned, one of the core building blocks required to get a dropwizard service
running in kubernetes, is a docker image that starts a dropwizard service. The pom file thus includes the configuration required to first build a jar containing
the dropwizard service and then build a docker image that includes this jar (and its dependencies).

Building the dropwizard jar is pretty standard and the following pom snippet shows how this is done:
{% highlight xml %}
<project>

    <plugins>

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

    ...

    </plugins>

</project>
{% endhighlight %} 

One difference between the recommended way of building a dropwizard jar (see [Building Fat JARs](http://www.dropwizard.io/1.2.2/docs/getting-started.html#building-fat-jars)) 
is that we do not build a fat jar. In the dropwizard documentation, the argument made for doing this is that building a fat jar makes it easier to promote the 
same artifact between repositories. This is true (and a great practise as opposed to say rebuilding for different environments), 
however in our case we will not be promoting jars between repositories but rather docker images, so we can easily include all dependencies outside of the jar 
but within the same image. A benefit of this is that we can make the build more efficient by avoiding the overhead of executing the [Shade Plugin](https://maven.apache.org/plugins/maven-shade-plugin/), 
as well as taking full advantage of docker layer caching by including all dependencies in their own layer. Doing this ensures that the final 
image layer (the layer that includes the dropwizard jar) is as small as possible. 



// we treat the docker image as the final artifact


I have chosen to not build a fat jar in this instance but rather include all dependencies in a  
 
 
 
 
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