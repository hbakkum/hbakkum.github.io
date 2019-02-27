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
frameworks (think Kubernetes, AWS ECS, Istio, Envoy, Dropwizard, Spring Boot, ...) that tempt development teams towards building microservices.

While I do believe the benefits of a microservices architecture can be real, the reality is the cost associated with implementing and supporting the architecture 
is usually very high. Because of this cost, I am skeptical that the majority of development teams will ever see a return on the investment that is required to build out a 
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
components can communicate via in-process function calls. For example, we could imagine component A exposes an external interface (e.g. a HTTP/REST interface) and in the context of processing a 
request to this interface has dependencies on component C and (indirectly) B.

As all components are still bundled into a single deployable artifact, making a change to any component will likely require a new deployment of the entire 
application. And while we can run many instances of the monolith application, doing so will require us to run all components in each instance.   

A microservices architecture takes the modularity seen in the monolith a step further by running components as independent processes, a.k.a microservices. In contrast to the monolith, each microservice will typically have its 
own source code repository with a dedicated build process consuming this to produce an artifact that can be deployed independently from the other services.

Communication between services is done via network calls, often (but by no means exclusively) using HTTP/REST or perhaps asynchronously through some 
messaging middleware (e.g. RabbitMQ). 

As services are more loosely coupled, it is possible to make and deploy a change to a service without having to redeploy any of the other services. 
In addition to this, we can also choose to run a different number of instances of each service (as seen in the diagram, services A and B have one 
instance each, while service B has two instances).

With these differences in mind, we can see that a microservices architecture can yield the following advantages over a monolith:

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

* I will be exceptionally brief in discussing each item. Full discussion of a single item could at least fill an entire blog post on its own. The point here is not a detailed explanation but rather to highlight that there is *a lot* to think about and do.

* While some of the challenges discussed will only be relevant in a microservices architecture, there are some that may also be present to some degree within a monoliothic architecture as well. However, you will find that as the number or services grow that the complexity of each challenge will also increase.

* Not all of these costs *have to* be realised and there may be strategies or compromises that can be made to avoid or mitigate some of them, particulary when you have a low number of services. That being said, to deal with just a subset of these issues is hard enough, and even if you manage to reduce or eliminate the effort on some, you still probably spent engineering hours in thinking and talking about it.

Ok, lets begin...
  
### Service Discovery ###
![Service Discovery]({{ site.url }}/assets/images/service-discovery.png){: .center-image }
As mentioned, microservices communicate with each other over the network and a service may need to call other services when processing a request. Such a service will therefore need to know about the network addresses for a service that it is to call. 

One option would be to hard-code these addresses into the services that need them. While this might work for a very small number of static services, it is not too difficult to see that as we increase the number of services that this solution quickly becomes unmanagable. Another problem with this solution is that it does not work particulary well when we have a dynamic environment. For example, we could imagine the addresses that we would want to use for a particular service changing due to:

* The number of instances of the service might change due to scaling events (and these could be manual or automated).
* An instance of the service might restart or terminate 
* A new service deployment might use different network addresses for the instances
* The service might be running but failing a healthcheck

To solve this problem we need to introduce a component into our architecture which is commonly referred to as a **service registry**. The service registry would contain a database of network addresses for each service, a mechanism for continuously updating the list of services and their addresses as these change, and of course a way to query for the addresses. Given we will often have multiple instances (and therefore addresses) for a given service, we also need a strategy for load balancing requests accross the available instances.

It is easy to see that the service registry is a fairly critical piece of infrastructure within the microservices architecture. Without it, services will not be able to discover each other which will poentially cause many requests to fail. It may be possible to tolerate brief outages to the service registry through the use of address caching and retry logic within your services (which you'll need to implement of course), but really this needs to be a highly available and well monitored piece of infrastructure.

And while there are tools out there to help prevent you from writing something from scratch (e.g. [KubeDNS](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/), [Consul](https://www.consul.io)), all of these still come with their own learning curve, integration work and operational overhead. 

### Error Handling ###
Given our application is now distributed over a network, there is many network related failures that we will be more susceptible to and will now need to handle.

For example, a service being called might be unavailable, timeout or crash while processing the request and the communications protocol being used may 
return many different types of results (e.g. if HTTP then 1xx -> 5xx status codes).  

You will need to decide what to do in these scenarios (e.g. retry, circuit breaker) and then implement this by writing code or configuration. And most likely
you will want to be alerted when the failure rate crosses some threshold.

### Healthchecks and Monitoring ###
More services means more healthchecks and processes that need to be monitored. Your probably going to want a central API to query all healthchecks
for a DC/node/service.

Further too this, some integration will be required between healthcheck monitoring and your service registry. Something needs to check if a service is
healthy and remove from the registry. Number of services running / monitor this and alert. 

### Logging ###
Your application logs will now be split across your services and you will need a way to centralise these. Further to this, requests made to your application
might now span multiple services so you will likely need a way to filter out logs for a particular request in order to debug activity. Typically this is
solved by allocating a request ID when the request is first received by the apop,livcation (e.g. on the public REST API) and this is then forwarded on to
any upstream services (e.g. by using a custom HTTP header). These services must then ensure that this ID is included in any logs it generates 
(as well as forwarding it on to any services it uses). 

### Security ###


### Performance ###

### Build and Deploy ###

### Testing ###

### API Compatibility ###
We no longer have the luxury of the compiler (in some cases) catching a broken interface...


# Summary # 
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

