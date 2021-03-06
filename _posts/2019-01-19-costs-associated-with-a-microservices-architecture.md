---
layout: post
title: "Costs Associated with a Microservices Architecture"
comments: false
description: ""
keywords: "microservices"
author: Hayden Bakkum
visible: 0
---
There is (still) a lot of industry hype surrounding microservices, and along with the hype, there is no shortage of shiny tools and 
frameworks (Kubernetes, AWS ECS, Istio, Envoy, Dropwizard, Spring Boot, ...) that tempt development teams towards building microservices.

While I do believe the benefits of a microservices architecture can be real, the reality is that the engineering costs associated with implementing and supporting the architecture 
are usually very high. Because of this cost, I am skeptical that the majority of development teams will ever see a return on the investment that is required to build out a 
fully fledged microservices architecture.  

Further to this, I think microservices can often serve as a major distraction for development teams, causing them to focus their efforts on building out
the architecture at the expense of more useful activities (e.g. quickly building out a new application as a monolith in order to first get market validation).  

So, rather than professing the benefits of microservices, in this article I'll explore some of the costs associated with the architecture in more detail.  

## Microservices: An Introduction ##
To begin with, I'll give a brief (and simplistic) overview of microservices for those readers that are unfamiliar with the architecture.

Its perhaps easiest to explain microservices by comparing them to a monolithic application as shown in the following diagram: 

![Monolith vs Microservices]({{ site.url }}/assets/images/monolith-vs-microservices.png){: .center-image }

A monolithic architecture is shown on the left. Here one would typically expect the source code for the application to be located in a single
source repository with a build process consuming this code to produce a single deployable artifact (e.g. a JAR or WAR file) which runs within a single process. 

Like any well designed application there will be some degree of modularity within the source code (as depicted by components A, B and C) and 
components can communicate via in-process function calls. For example, as depicted in the diagram, we could imagine component A exposes an external interface (e.g. a HTTP/REST interface) and in the context of processing a 
request to this interface has dependencies on component C and (indirectly) B.

As all components are still bundled into a single deployable artifact, making a change to any component will likely require a new deployment of the entire 
application. And while we can run many instances of the monolith application, doing so will require us to run all components in each instance.   

A microservices architecture takes the modularity seen in the monolith a step further by running components as independent processes, a.k.a microservices. In contrast to the monolith, each microservice will typically have its 
own source code repository with a dedicated build process consuming this to produce an artifact that can be deployed independently from the other services.

Communication between services is done via network calls, often (but by no means exclusively) using HTTP/REST or perhaps asynchronously through some 
messaging middleware (e.g. RabbitMQ, Google PubSub). 

As services are more loosely coupled, it is possible to make and deploy a change to a service without having to redeploy any of the other services. 
In addition to this, we can also choose to run a different number of instances of each service (as seen in the diagram, services A and B have one 
instance each, while service C has two instances).

With these differences in mind, it can be argued that a microservices architecture may yield the following advantages over a monolith:

* Smaller services are easier for developers to understand and faster to build/test/deploy, so feedback loops in the development process are tightened

* Services can be deployed independently, making it easier for multiple development teams to release smaller changes at their own cadence

* Services can be developed in different programming languages and frameworks so you can pick the best tool for the job at hand

* Specific services can be scaled out as needed, requiring only additional resources to host the service being scaled as opposed to the entire application

* Failure of a service need not bring down the entire application and may result in only a small piece of functionality being affected 

## But at what cost? ##
The advantages of microservices are compelling, but the reality is there is *significant* engineering work required to support such an architecture. As the number of services grow you will find yourself needing to build an ecosystem that ensures these services can be:

* Connected together in a reliable and secure way
* Built and deployed independently without breaking API contracts between services
* Easy to debug and monitor

For the remainder of this article I'll attempt to itemize some of the engineering challenges you will encounter. But before this, a few points to mention:

* I will be reasonably brief in discussing each item. Full discussion of a single item could at least fill an entire blog post on its own. The point here is not a detailed explanation but rather to highlight that there is *a lot* to think about and do.

* While some of the challenges discussed will only be relevant in a microservices architecture, there are some that may also be present to some degree within a monolithic architecture as well. However, you will find that as the number or services grow that the complexity of each challenge will also increase.

* Not all of these costs *have to* be realised and there may be strategies or compromises that can be made to avoid or mitigate some of them, particularly when you have a low number of services. That being said, to deal with just a subset of these issues is hard enough, and even if you manage to reduce or eliminate the effort on some, you still probably spent engineering hours in thinking and talking about it.

So let's begin...
  
### Service Discovery ###
![Service Discovery]({{ site.url }}/assets/images/microservices-service-discovery.png){: .center-image }
As mentioned, microservices communicate with each other over the network and a service may need to call other services when processing a request. 
Such a service will therefore need to know about the network addresses of the service(s) that it is to call. 

One option would be to hard-code these addresses into the services that need them. While this might work for a very small number of static services, 
it is not too difficult to see that as we increase the number of services that this solution quickly becomes unmanageable. Another problem with this solution is 
that it does not work particularly well when we have a dynamic environment. For example, we could imagine the list of valid addresses for a service changing due to:

* An instance of the service might terminate
* An instance of the service might restart and in some environments the same network address might not be used
* Along these same lines, a new deployment of the service might not assign the same network address to the new instances 
* The number of instances of the service might change due to scaling events (this could be manual or automated)
* The service might be running but failing a healthcheck

To solve this problem we need to introduce a component into our architecture which is commonly referred to as a **service registry**. The service registry would contain a database of network addresses for each service, a mechanism for continuously updating the list of services and their addresses as these change, and of course a way to query for the addresses. Given we will often have multiple instances (and therefore addresses) for a given service, we also need a strategy for load balancing requests accross the available instances.

It is easy to see that the service registry is a fairly critical piece of infrastructure within the microservices architecture. Without it, services will not be able to discover each other which will likely cause many requests to fail. It may be possible to tolerate brief outages to the service registry through the use of address caching and retry logic within your services (which you'll need to implement of course), but really this needs to be a highly available and well monitored piece of infrastructure.

And while there are tools out there to help prevent you from writing something from scratch (e.g. [KubeDNS](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/), [Consul](https://www.consul.io)), all of these still come with their own learning curve, integration work and operational overhead. 

### Error Handling ###
![Error Handling]({{ site.url }}/assets/images/microservices-error-handling.png){: .center-image }
We can see that given our microservice application will consist of many services that interact across a network, that we will probably have more network nodes and more network chatter versus the monolith. We will therefore likely find 
ourselves more susceptible to node and network failure. These failures are inescapable - they will at some point happen, even if briefly - and these will result in, for example, connection resets, connection/read timeouts and 
partial responses. So in the absence of any retry logic, we might find ourselves less robust than the monolith. 

To complicate things further, the communications protocol used between services may come with its own set of error codes and semantics. For example, HTTP supports error codes 1xx through to 5xx as well as a number of different
request methods. But we can't just use a blanket rule to retry everything. For example, does it make sense to retry when the response is in the 4xx range which indicates a client error? In many cases no, for example, if we get an HTTP
status of 400 back from the server, this suggests the request is malformed and so if we retry, the request will likely just fail again. Also, if the failed request is not idempotent (e.g. an HTTP POST), then probably its not safe to retry the operation for risk of a double submission.  

Further to this, we might find that only a single instance of a service is in error while the rest of the instances are fine. Does it make sense to continue sending this instance requests? If it is timing out, then this will
be contributing to some degradation in performance for some users, or it could be that if we stopped sending this instance traffic for a bit that it will recover on its own. This scenario is another aspect that our error 
handling logic may want to consider.   

So in summary you will need to decide what to do in these error scenarios (e.g. fail fast, retry/backoff, circuit breaker) and then implement this by writing code or configuration. Again, there are tools that can help 
(e.g. [resilience4j](https://github.com/resilience4j/resilience4j)), but nothing comes for free - you will still need to invest time in learning these, agreeing on and implementing your error handling policies and 
testing the scenarios. 

### Healthchecks and Monitoring ###
![Healthchecks]({{ site.url }}/assets/images/microservices-healthcheck.png){: .center-image }
Sure, we will need these too for a monolith, but this will be a more complex problem in a microservices architecture. 

For example, we could imagine that in a simple monolithic application requests to the application would first hit a load balancer which would then distribute requests across a fixed pool of application instances. 
The load balancer could also check the health of each instance and depending on the health status of each instance, would either add or remove it from the pool. 

However in the microservices world our monolith will be split across multiple services, with each service potentially consisting of a dynamic number of instances and 
the additional complexity that a request to the application could also span more than one service. So you will need a mechanism to not only check the health of each service instance, but to also add or remove a 
service instance from the list of available instances that can satisfy a request based on its health status.

It is easy to see that in fact their will be tight integration between the healthcheck mechanism and your service registry as your healthcheck may need to use your service registry to get a complete
list of all available service instances and may also need to update the service registry when a service instance healthcheck status changes.

In addition to this, each service will likely have its own healthcheck definitions and so you will need to ensure there is a consistent way for developers to define and register healthchecks for the services they are responsible for. 

A final feature you will probably want is an API that can be used to query the healthcheck status of all service instances. This can then be used by your monitoring system to raise alerts.
It will also be very useful in debugging issues. For example, the healthcheck API my help you isolate the cause of an application failure as the response could show that the failures correspond to a failed datacenter or particular cloud service. 

So in summary, your microservices ecosystem will likely need:

* A mechanism to discover all (both healthy and unhealthy) service instances in your environment
* A means for each service to define and register what its healthchecks are 
* A mechanism for running the healthchecks against each service periodically and possibly on demand
* A way to update the list of service instances in the service registry that are available for satisfying a request (i.e. healthy services will be included in this list and unhealthy services will be removed)
* An API to query the current health status of service instances

### Logging ###
Your application logs will now be split across your services and you will need a way to centralise these. Further to this, requests made to your application
might now span multiple services so you will likely need a way to filter out logs for a particular request in order to debug activity. Typically this is
solved by allocating a request ID when the request is first received by the apop,livcation (e.g. on the public REST API) and this is then forwarded on to
any upstream services (e.g. by using a custom HTTP header). These services must then ensure that this ID is included in any logs it generates 
(as well as forwarding it on to any services it uses). 

### Security ###
SSL, higher exposed API surface area, JWT

### Performance ###

### Build and Deploy ###
Conventiosn, automation, how do we deploy multiple service changes (as aall componenents released at once on a monolith, then this is easier) - are we comfortable releasing like this for microservices

maven archetype for example

which nodes can you to deploy (k8s can solve).

### Testing ###
API contract testing + infrastructure
Starting services harder for manual testing
Chaos testing
What happens during rolling deployment (easier with monmolith - one application).

### API Compatibility ###
We no longer have the luxury of the compiler (in some cases) catching a broken interface...


# Summary # 
As you can see, you can build a lot and still have nothing that provides any value to your customers. 

None of the problems listed are insormountable and many diffrent techniques and tools exist to help solve these. But even so, the point is that addressing even
a subset of these problems requires implementation of said patterns and knowledge building and integration with said tools which quickly becomes a significant 
engineering effort to implement and continouslky maintain these solutions. And in addition to this, I think its debatable whether most teams can actually get a 
return on their investment when it comes to microservices - most of us are not operating at Netflix or Amazon scale and even if we are, there are companies 
managing with a monolith just fine e.g. facebook.    

Particulary when starting a new development, I think all the tooling, frameworks and blog posts advocating for microservices serve as a major distraction when
really one should focus on validating their application first. Therefore I completely agree with the MonoltihFirst approach and it should be pointed out
that even advocates of microservices started out with a monolith. By doing this, we also delay 
any decomposition until after we have actually observeds the application running in production. This then places us in a better position to make decisions about
which components might need breaking off based on empircal evidence as opposed to assumpotions. And you may just come to the conclustion that decomposition
is not required anyway, saving yourself a huge invesntment in an unnecessary architecture.

If you do find yourself in need of breaking a monolith. For example, perhaps it possible to break off a component that has no dependency on the remaining
monolith, potentially   

# Further Reading #


Kubernetes for example can help with this problem as it enables a developer to define healthcheck definitions inside of a manifest file for your microservice (pod?) which defines a 
healthcheck that is custom to your service. Kubernetes will then periodically execute this healthcheck and update the status of each pod in 


 provides an API for discovering all running service instances (i.e. pods) in a cluster,

dynamic nodes too

as we have seen in the section on service discovery,
the number and network addresses of your microservices can be dynamic and so your healthcheck will need to take this into account. In addition to this, the number of available instances for
a service will fluctuate as service instances either fail their healthchecks or recover from a previous fail.  


You may have many different services accross multiple
developement teams. Ideally you will want teams developing microservices to be able to declare the checks they want to occur on the service without having to write or manage the checking infrastructure themselves.

For example, in kubernetes each service can define a healthcheck in a manifest file. If this fails for any instances, they will be removed from KubeDNS, etc. There is the k8s api to query
for failures, but not much info re specific failure message.



Multiple teams developing services independently - need to define conventions so each can integrate their healthchecks/metrics into the ecosystem
Don't want to manually check these
Need tight integration with service registry to a) find available services, b) update their status which may have the effect of taking them in/out of service.
Need to expose API for a) monitoring software to query and b) developers to query
Need a way to registry healthchecks in a consistent way
Also need a way to monitor for errors in the system e.g. http error code counts etc.
Define common way for scraping metrics from services

More services means more healthchecks and processes that need to be monitored. Your probably going to want a central API to query all healthchecks. Can see for example if failing 
healthcecks limited to a DC or particular cloud service.
for a DC/node/service.

Further too this, some integration will be required between healthcheck monitoring and your service registry. Something needs to check if a service is
healthy and remove from the registry. Number of services running / monitor this and alert. 