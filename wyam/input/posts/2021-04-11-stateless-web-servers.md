Title: Stateless Web Servers
Published: 2021-04-11
Tags: [.NET, Redis, SignalR]
---  

It all began when we discovered a strange behavior in one of our applications - users had been randomly logged out of the system.  

## The Problem  

The system in question has multiple instances of web servers serving the UI and a load balancer with sticky connections configuration. We were not able to reproduce the issue in any of our environments and not all customers were affected. After a thorough investigation, it appeared that this was specific to some ISPs changing IPs and breaking the "stickiness" of connections.  

## Load Balancers Sticky Connections  

Why were sticky connections selected as a load balancing strategy, to begin with? Probably because it had been a fast solution to a problem at the time being. With the rapid growth of the customer base and team racing to deliver features, no one was willing to give it a second thought. Sticky connections offer the promise that users would hit the same web server they connected the first time, so from a developer's point of view, it meant simplicity - the same as having only one web server.  

## Web Servers State  

Why was connecting to the same web server important? Because there was an application state, namely the user session, stored in the web server's memory. [SignalR](https://docs.microsoft.com/en-us/aspnet/signalr/overview/getting-started/introduction-to-signalr) also used the session and the transport protocol was fixed as long polling.

## Choosing a Solution - Stateless Web Servers  

The problem could be solved by an even more sophisticated load balancer configuration. We felt, however, that increasing the infrastructure complexity will harm scalability in the long run. We decided to go with stateless web servers instead. Then a simpler and more efficient load balancing strategy, like least connections or round-robin, can be implemented. Now there is a catch - stateless web server does not mean that application does not have a state, it means that the state will not be bound to the web server's memory thus enabling any webserver to serve a request.  

## Achieving "Statelessness"  

We needed to solve two application-level problems - moving the session state out of memory and ensuring SignalR was still working. The first step was to decide where the session state would be persisted. We researched three options - [Microsoft Session State Server](https://docs.microsoft.com/en-us/previous-versions/aspnet/ms178586(v=vs.100)#state-server-mode), [Microsoft SQL Server](https://docs.microsoft.com/en-us/previous-versions/aspnet/ms178586(v=vs.100)#sql-server-mode), and [Redis](https://docs.microsoft.com/en-us/azure/azure-cache-for-redis/cache-aspnet-session-state-provider). All of the options were easy to implement requiring a NuGet package dependency and configuration. Of course, they also required their specific service to run on a separate server, accessible from the web servers. Session State Server was dismissed as we were not sure how much support it will get in the future. MS SQL Server seemed like a too heavyweight solution, even using memory-optimized tables. We chose Redis because it gave us the best performance and we had more use for it in mind. Scaling SignalR boiled down to configure it to use [Redis Backplane](https://docs.microsoft.com/en-us/aspnet/signalr/overview/performance/scaleout-with-redis).  
It is worth noting that any other resources bound to a server should be considered and moved to a shared location (for example - files on the local files system).  

## Implementation  

### Prerequisites  

During the proof of concept phase, we had a glimpse of the ["binding redirect hell"](https://nickcraver.com/blog/2020/02/11/binding-redirects/) that was awaiting us. Targeting a bit older version of .NET Framework and having a ton of dependencies was not the ideal situation to be in, especially when you throw in the mix some transient dependencies on .NET Standard. After going through verbose MS Build logs countless times we decided that a move to .NET Framework 4.8 and SignalR 2 and updating and cleaning up dependencies will be best in the long term. The decision was reinforced by the fact that 4.8 is the last stop for the "full" framework and since we were not able to move to .NET Core, at least we could be on the latest and long-term supported version.   

### Session State

Once upgrades were completed we moved to get the session state out of the server memory. We opted for the built-in binary serialization, meaning we needed to add Serializable attributes to a bunch of classes. Since the session state was used to put in whatever you want, it was abused to be somewhat of a cache for various stuff. We were surprised to discover that when serialized in some cases it can be up to 3MB! Imagine this serialized and moved over the network on every request. It took some work to trim it down to a more manageable size - around 100KB in worst cases. This started another long-term project which goal was to have a session state containing only a few strings and a bunch of ids, and a distributed cache to store frequently used data - but this is another story.  
One important piece of configuration was to use the same encryption configuration and keys on all IIS instances. Since an auth cookie produced by one server will eventually end up sent to a different one, each web server should be able to understand it.   

### SignalR 

Scaling out SignalR with Redis was easy, again it took a NuGet package dependency and configuration. There were two caveats though - we had to explicitly set the protocol to WebSocket for all clients which in turn allowed skipping protocol negotiation. This, and the upgrade to SignalR 2, meant that the server session was not supported. It was not a big problem since the requests were still authenticated and we could set up a few parameters to be passed with each request. On the server, they were verified and then used to recreate the session object getting whatever was needed from a cache. We also replaced custom notifications filtering involving storing connections and users with SignalR groups.  
We had windows services producing notifications that were sent to web servers via custom messaging, and in turn, web servers used SignalR to broadcast to clients. We were able to hook up our windows services to the SignalR directly leveraging SignalR backplane and groups. This greatly simplified the overall implementation.  

### Roll Out  

Getting all this to production without disturbing customers could be tricky. We planned to do it in phases and have a few weeks between each phase deployment to root out and fix eventual problems. We also had a staging environment with a properly configured load balancer with the envisioned future production configuration.  

The first phase was the riskiest - migration to .NET Framework 4.8, SignalR 2, and WebSockets. There were some problems with WebSockets connections, generally related to customer's infrastructure. We found this https://websocketstest.com to be quite handy when looking for connectivity issues, kudos to its creators and maintainers.  

The second phase was switching SignalR to use the Redis backplane. It was only a configuration so we had a fast rollback option. We had also implemented an Azure Service Bus backplane as an alternative if something went wrong. Both options were not used since the switch went without problems.  

Next was switching windows services to use the Redis backplane for broadcasting notifications, again without problems. This was a configuration too so a rollback to the custom solution was a matter of configuration.  

The third phase was moving the session state in Redis. Once more it was a matter of changing configuration. This went smoothly without any issues at all.  

All this done, there was the last and most important piece of the puzzle to complete the solution - changing the load balancer configuration. Because all of the required configurations were already in production and working as expected we were quite confident that will be without any problems - which was the case in the end.  

This completed the implementation of our solution and it solved the original problem, and as an additional benefit improved the scalability of our web servers.  

## Conclusion  

A move from stateful to stateless web servers is not easy, mostly because the software is often implemented with the assumption that all requests for a given session will be handled by the same server. With careful planning, implementation, and rollout, it is achievable with minimal disturbance of service. Stateless web servers are easier to scale and manage and applications are more resilient to crashes or restarts. In my case, this also surfaced some hidden problems and improved the system as a whole. 
