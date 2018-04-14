---
layout: post
title: "Creating a Basic Dropwizard Microservice in Kubernetes"
comments: true
description: ""
keywords: "kubernetes dropwizard microservices"
author: Hayden Bakkum
visible: 1
---

In this post I'll describe how to create a basic dropwizard based microservice and deploy this into a kubernetes cluster. For the purposes
of this demo, I'll be using [Google's Kubernetes Engine](https://cloud.google.com/kubernetes-engine/) as the kubernetes cluster provider.

In order to get a dropwizard service deployed to kuberenetes, a few basic building blocks are required:
* a docker image that starts a dropwizard service
* a kubernetes configuration file that references said image and describes how it should be deployed to the cluster (see [kubernetes deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/))
* a build pipeline to trigger the image build and then manage the deployment to kubernetes  

The service used as an example will expose a REST endpoint for returning a list of movies and will be aptly named the **movies-service**. 
Full source code for this service can be found [here](https://github.com/hbakkum/movies-service).

### Building a Dropwizard Docker Image ###

To begin with, we'll look at the services pom file in more detail. As previously mentioned, one of the core building blocks required to get a dropwizard service
running in kubernetes, is a docker image that starts a dropwizard service. The pom file thus includes the configuration required to first build a jar containing
the dropwizard service and then build a docker image that includes this jar (and its dependencies).

Building the dropwizard jar is pretty simple and the following pom snippet shows how this is done:
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
is that we do not build a fat jar. In the dropwizard documentation, the argument made for building a fat jar is that it is easier to promote a single artifact
between environments. This is true (and a great practise as opposed to say rebuilding for different environments), 
however in our case we will not be promoting jars between environments but rather docker images, so we can easily include all dependencies outside of the jar 
but within a single image. A benefit of this is that we can make the build more efficient by avoiding the overhead of executing the [Shade Plugin](https://maven.apache.org/plugins/maven-shade-plugin/), 
as well as taking full advantage of docker layer caching by including all dependencies in their own layer. Doing this ensures that the final 
image layer (the layer that includes the dropwizard jar) is as small as possible. 

You should also be able to see that we have added a classpath entry to the dropwizard jar that we will use as the location for its dependencies. Looking
further ahead in the pom the maven dependency plugin is used to copy these dependencies into the target directory so they can then be copied into the
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

Finally we build the docker image. To do this the [Spotify Dockerfile Maven Plugin](https://github.com/spotify/dockerfile-maven) is used. By default, this plugin looks for 
a Dockerfile in the projects root directory and uses this to run a docker build. 
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
the docker image and a **mvn clean deploy** will produce the image as well as push it to a docker registry. For this example, the docker registry being
used is [Google's Container Registry](https://cloud.google.com/container-registry/). 

As mentioned, the plugin will look for a **Dockerfile** in the same directory as the pom file and the contents of this Dockerfile are as follows:
{% highlight docker %}
FROM openjdk:8-jre

# declare that the container listens on these ports
EXPOSE 8080
EXPOSE 8081

# standard command for starting a dropwizard service
ENTRYPOINT ["/usr/bin/java", "-jar", "/usr/share/movies-service/movies-service.jar", "server", "/usr/share/movies-service/config.yaml"]

# add in project dependencies
ADD target/lib /usr/share/movies-service/lib

# add dropwizard config file - the server is configured to listen on ports 8080 (application port) and 8081 (admin port)
ADD target/config/dw-config.yaml /usr/share/movies-service/config.yaml

# add built dropwizard jar file - the JAR_FILE argument is configured in the dockerfile maven plugin 
ARG JAR_FILE
ADD target/${JAR_FILE} /usr/share/movies-service/movies-service.jar
{% endhighlight %}

Thus running a **mvn clean deploy** will build our dropwizard image and push it to container registry, ready for deployment to kubernetes.

And one final note on the build - the maven deploy plugin has been configured to skip deployment of the dropwizard jar to a maven repository as
my final build artifact is the docker image which has already been pushed to the container registry.

### Creating Kubernetes Configuration ###
The next thing we need to do is produce the kubernetes configuration file. To do this we create a yaml file which contains the kubernetes resources we wish
to create, in this case, a kubernetes *deployment* and *service*. 

Our docker image will be contained within a *pod*, which is a group of one or more containers running as a single deployable unit within kubernetes. 
The deployment defines the manner in which these pods should be deployed across the cluster. For example, the deployment declares how many pods should be run, 
what containers should be running within each pod, their update strategy, etc. For more information, see kubernetes 
[pods](https://kubernetes.io/docs/concepts/workloads/pods/pod/) and [deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/). 

A kubernetes [service](https://kubernetes.io/docs/concepts/services-networking/service/) is a construct that can be used to select a set of pods so that they may be accessed as a single logical unit. For example,
we can configure the service to select our movies service pods. By default, the kubernetes service will assign this set of pods a virtual IP (VIP) (this VIP
is also only resolvable from within the cluster). The pods can then be accessed over this VIP and kubernetes will perform some rudimentary load balancing 
across the pods as well as ensuring pods are added/removed from the VIP if they become available/unavailable.

The yaml file containing these resources is located in **src/main/config/k8s-config.yaml**:
{% highlight yaml %}
kind: Service # kubernetes service definition
apiVersion: v1
metadata:
  name: movies-service
spec:
  ports:
  - name: http
    port: 80
    targetPort: http-app
    protocol: TCP
  selector: # this service selects all pods with the label, app=movies-service
    app: movies-service
---
kind: Deployment # kubernetes deployment definition
apiVersion: extensions/v1beta1
metadata:
  name: movies-service
spec:
  replicas: 2 # the number of pods we wish to run
  revisionHistoryLimit: 5
  strategy:
      rollingUpdate: # controls update strategy
        maxSurge: 1
        maxUnavailable: 1
      type: RollingUpdate
  template:
    metadata:
      name: movies-service
      labels:
        app: movies-service # this label is used to select the movies-service pods in the service definition above
    spec:
      containers: # list of containers to run within the pod. We'll run just a single container, i.e. our dropwizard based movies service
      - name: movies-service
        image: us.gcr.io/${gcloud.project}/movies-service:${project.version} # the image name of the movies service. The property placeholders get resolved at build time
        imagePullPolicy: IfNotPresent
        readinessProbe: # these probes are used to indicate whether or not the pod is ready to receive traffic
          httpGet: # make an http GET to the dropwizard healthcheck endpoint, if we get a 2xx back (i.e. all checks passed), then this pod will receive traffic
            path: /healthcheck
            port: 8081
        ports: # the ports that we want exposed on the IP assigned to the pod
        - name: http-app # the dropwizard application port
          containerPort: 8080
        - name: http-admin # the dropwizard admin port
          containerPort: 8081
{% endhighlight %}  

It should also be highlighted that this file is in fact a template that is passed through maven resource filtering so that 
it gets merged with maven build properties. The resulting file is output to **target/config/k8s-config.yaml** and this is what
will actually get used in the build pipeline to deploy our movies-service to kubernetes.

### Deploying the Dropwizard Service to Kubernetes ###
And finally we need to define a build pipeline for coordinating the build and deployment to kubernetes. For this, 
I'll be using Jenkins. The following Jenkinsfile defines the pipeline for build and deployment of the movies-service:
{% highlight groovy %}
// This needs to run on a jenkins slave with gcloud installed as well as a key file for a gcloud service account 
// that has permission to push images to container registry and deploy to kubernetes.
//  
//  The variables 'GCLOUD_PROJECT' and 'K8S_CLUSTER' are provided via a parameterized build
//
node("gcloud") {
    stage("clean") {
        deleteDir()
    }

    stage("checkout") {
        git url: "git@github.com:hbakkum/movies-service.git"
    }

    // Build the dropwizard docker image and upload this to container engine. The env variable 
    // 'DOCKER_GOOGLE_CREDENTIALS' points to the location of our gcloud service account key and 
    // is used by the spotify dockerfile plugin to authenticate to container registry
    stage("build") {
        withEnv(["GCLOUD_PROJECT=${GCLOUD_PROJECT}", "DOCKER_GOOGLE_CREDENTIALS=/var/jenkins/gcloud-accounts/${GCLOUD_PROJECT}-jenkins-slave-account.json"]) {
            sh "mvn clean deploy"
        }
    }

    stage("deploy") {
        // Configures kubectl with kubernetes cluster credentials and endpoint information
        sh "gcloud auth activate-service-account jenkins-slave@${GCLOUD_PROJECT}.iam.gserviceaccount.com --key-file /var/jenkins/gcloud-accounts/${GCLOUD_PROJECT}-jenkins-slave-account.json"
        sh "gcloud --project ${GCLOUD_PROJECT} container clusters get-credentials ${K8S_CLUSTER} --zone us-central1-a"
        
        // Apply the templated kubernetes config file containing the deployment and service, then wait for the rollout to complete
        sh "kubectl apply -f target/config/k8s-config.yaml"
        sh "kubectl rollout status deployment/movies-service"
    }
}
{% endhighlight %}

After a successful run in jenkins, we see the following output:

{% highlight bash %}
[INFO] Image bb7e6490d88d: Pushed
[INFO] 0.17: digest: sha256:699041a8ba20eed14eaf73f8bee737168291471ab24c87a732b4a28b3eded79c size: 2626
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time: 50.881 s
[INFO] Finished at: 2018-03-09T04:21:36+00:00
[INFO] Final Memory: 46M/376M
[INFO] ------------------------------------------------------------------------
[Pipeline] }
[Pipeline] // withEnv
[Pipeline] }
[Pipeline] // stage
[Pipeline] stage
[Pipeline] { (deploy)
[Pipeline] sh
[movies-service-pipeline] Running shell script
+ gcloud auth activate-service-account jenkins-slave@${GCLOUD_PROJECT}.iam.gserviceaccount.com --key-file /var/jenkins/gcloud-accounts/${GCLOUD_PROJECT}-jenkins-slave-account.json
Activated service account credentials for: [jenkins-slave@${GCLOUD_PROJECT}.iam.gserviceaccount.com]
[Pipeline] sh
[movies-service-pipeline] Running shell script
+ gcloud --project ${GCLOUD_PROJECT} container clusters get-credentials ${K8S_CLUSTER} --zone us-central1-a
Fetching cluster endpoint and auth data.
kubeconfig entry generated for blog-cluster.
[Pipeline] sh
[movies-service-pipeline] Running shell script
+ kubectl apply -f target/config/k8s-config.yaml
service "movies-service" created
deployment "movies-service" created
[Pipeline] sh
[movies-service-pipeline] Running shell script
+ kubectl rollout status deployment/movies-service
Waiting for rollout to finish: 0 of 2 updated replicas are available...
Waiting for rollout to finish: 1 of 2 updated replicas are available...
deployment "movies-service" successfully rolled out
[Pipeline] }
[Pipeline] // stage
[Pipeline] }
[Pipeline] // node
[Pipeline] End of Pipeline
Finished: SUCCESS
{% endhighlight %}

To confirm the deployment was successful, we can list out the pods (in this example, I configured the kubernetes deployment
to contain two pods) the kubernetes cluster:
{% highlight bash %}
[user@host] kubectl get pods
NAME                              READY     STATUS    RESTARTS   AGE
movies-service-4edf164fcd-ab3ss   1/1       Running   0          29m
movies-service-4edf164fcd-3455w   1/1       Running   0          29m
{% endhighlight %}

We can also check that a kubernetes service exists:
{% highlight bash %}
[user@host] kubectl get service movies-service
NAME             TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
movies-service   ClusterIP   10.55.244.103   <none>        80/TCP    32m
{% endhighlight %} 
 
And finally, from within one of our pods, we can make a curl request to the movies service:
{% highlight bash %}
# first, open up a shell on one of our movie pods
[user@host] kubectl exec -it movies-service-4edf164fcd-ab3ss /bin/bash

# Now we have a shell on the pod, we can make requests to our dropwizard services via the movies service cluster IP. 
# These curl requests get load balanced across the two pods 
[user@pod] curl "http://10.55.244.103/movies"
[{"id":"1","name":"The Matrix"},{"id":"2","name":"The Terminator"}]

# Kubernetes also registers a DNS entry for our service which resolves to the cluster IP. This gives us a nice
# service discovery mechanism
[user@pod] curl "http://movies-service/movies"
[{"id":"1","name":"The Matrix"},{"id":"2","name":"The Terminator"}]
{% endhighlight %}

Of course, although accessing the service in this way is fine for service to service communication within a kubernetes cluster,
at some point we'll want to expose our service outside of the cluster. More on that in an upcoming post...