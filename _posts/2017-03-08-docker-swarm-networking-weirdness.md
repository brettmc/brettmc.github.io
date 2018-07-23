---
title: Docker swarm networking weirdness
---

I've got a big project deployment coming up. It's fully containerised with docker, the only external dependency being an Oracle database.

We've been using the excellent docker-compose tool to spin up our development stack on  developer workstations, the core components of which are:

* a node.js app which acts as a websocket server using socket.io (an ios/android app will connect to this)
* an activeMQ message queue, which passes messages back and forth from the websocket server to various "worker nodes"
* worker nodes. A whole lot of PHP cli containers running react/stomp, which perform all of the actual application logic, such as:
- authenticate a user via an oauth token
- translate a voice/text command into an intent
- route messages to queues based upon a workflow
- process messages, to generate a response for the user

This has worked spectacularly well in development, and I've really loved being able to log messages to stdout from each container, then watch the combined container logs to see each step as a message bounces around through various containers then back to the client. With so many moving parts, I'm not sure how I would have been able to visualize message flows otherwise.

Now I've got two nice, shiny, 3-node swarm clusters. All for me. Bleeding edge, upgraded to docker engine 1.13.1 within a couple of days of its release.

The options for deploying this multi-container app onto the swarm that I have considered are:
* docker stack deploy
  - a nice logical grouping of services which are treated as a single entity
  - can apply rolling updates across the stack, upgrading containers over time so that there should be no loss of service
  - can deploy a stack based on a "distributed application bundle" which you can generate from a compose file. This feature is experimental, so we won't use it in production.
  - can deploy a stack based on a compose file. This sounds like the best option, however it doesn't currently understand environment variables or env files, which are one of the features I really love about compose. This probably means I'll end up writing a script to process a templated compose file, doing variable substitution as appropriate for a test/production configuration.
* docker services
  - pretty similar to stack, but slightly lower level. You get to control startup order of your containers (since you have to write a script to start all your services). That shell script can be made to understand variable substitution, though.
 	- you need to start and stop services individually (or via a script), and if there are enough services running, you're probably going to be relying on naming convention to help you work out which services belong together
* docker-compose
- old faithful. It hasn't let me down yet. But it doesn't work with swarm as well as it could - you can only deploy your code onto a single node, which isn't great from a redundancy/failover perspective - if that node goes down, you're toast. Of course, you could have as many stacks as you do nodes, and a failover mechanism in place. This will be my fallback method of nothing else works satisfactorily.

Once I was able to deploy my application onto the swarm, it was immediately clear that the application was not working as intended. Workers were not always receiving messages off the message queue, even though I am pretty sure they are there.

The first few things I tried when troubleshooting this were:
* deploy only one instance of each service, and just run the minimum amount of services to see the problem (in this case, message queue, websocket server, and authentication worker)
* ensure each service is running on the same node (via placement constraints in the compose file)
* run the application as individual services instead of a stack

None of these seemed to help, so I suspect it's related to the overlay network used in swarm. If I deploy the application (using the exact same images) via docker-compose onto a swarm node, everything works perfectly.

I next investigated whether the stomp connection from the authentication worker was being silently closed. I've written a separate blog post on how I investigated that one, and having patched the stomp client so that my code can complain loudly if connectivity is lost (ie the TCP connection is terminated). That doesn't happen, so I'm thinking that it must be getting killed somewhere in the network layer. So if a connection is established to the message queue, and it's not being lost, but no messages seem to be getting through, what next?

My next suspect was our corporate firewall. I've previously seen cases where it snipes TCP database connections that have been idle for a while, with neither the database or client being aware that this has happened until one of them tries to communicate. That's also not the case, as the firewall does not currently operate between the swarm nodes.

So I still think the overlay network is somehow responsible, but maybe I don't need to know *why* it happens. The fact that a long-running TCP connection *can* get disconnected randomly with neither side knowing that it has happened means that my code should assume that it *will* happen. A good way to prevent or quickly identify dead TCP connections is with a heart beat. Client, or server, or both periodically send a message to each other. If either side fails to receive a message within the agreed period, then it should assume that the connection has failed, and take appropriate action (eg a client might attempt to reestablish the connection). A bonus side-effect of this, I believe, is that the low-volume but regular "chatter" over the TCP connection makes it less likely that something will identify the connection as idle and terminate it.
