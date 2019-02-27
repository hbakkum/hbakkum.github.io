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

* I will be exceptionally brief in discussing each item. Proper discussion of a single item could at least fill an entire blog post on its own. The point here is not a detailed explanation, rather, to highlight that there is *a lot* to think about and do.

* While some of the challenges discussed will only be relevant in a microservices architecture, there are some that may also be present in some degree within a monoliothic architecture too. However, generally you will find that as the number or services grow the complexitiy of each challenge will grow.

* Not all of these costs *have to* be realised. There may be strategies or comprimises. For example, you may get away with hard coded addresses for a smaller number of services, but as you add more services, this wil quickly become unmanagable.

Ok, lets begin...
  
### Service Discovery ###
![Service Discovery]({{ site.url }}/assets/images/service-discovery.png){: .center-image }
Microservices communicate with each other over the network and a service may need to call other services when processing a request. When a service needs to call another service it will therefore need a way to discover the network addresses of any services it depends on. Further to this, it is often the case that the number of services
running is dynamic, services may come and go as they are auto scaled, restarted or perhaps when passing/failing a healthcheck. 

To solve this we need to introduce some form of service registry that can be queried for the list of addresses for a target service. It is also important that
this registry is continuously updated as the number of available services changes. 

Given the critical nature of this, it is also important that we invest time in ensuring that it is highly available and well monitored. 

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



These microservices interact with each other via loosly coupled service interfaces (e.g. HTTP/REST). Changing one
serice does not require releasing all services and we can run a different number of copies of service.

would take this source code and build a single deployable artifact (e.g. a JAR or WAR file). Like any 
well designed application there will be some degree of modularity within the source code and this is depicted with components A, B and C.
These components interact with each other via in-process function calls. For example, in Java these dependencies could be via method calls on interfaces.
If we wanted to release a change to a single component, this would require deploying the entire application. And while we can run multiple copies of the 
application, this does require us to run all components.     

A microservices architecture takes the modularity seen in the monolith a step further by splitting components into independent processes, a.k.a microservice. 
In contrast to the monolith, each microservice would have its code in its own repository with a dedicated build process taking this source code and building
a independnelyt deployable artifact. These microservices interact with each other via loosly coupled service interfaces (e.g. HTTP/REST). Changing one
serice does not require releasing all services and we can run a different number of copies of service.

Taking these differences into account, microservices can yield the following advantages over a monolith:
