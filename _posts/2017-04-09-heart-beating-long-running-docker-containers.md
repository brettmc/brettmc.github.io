---
title: heart-beating from long-running docker containers
---

In my recent experience with docker containers and long-running processes, I've started to gain an understanding of what a big, scary world it is out there for a little TCP connection. They can get killed by various things within the networking layer (particularly if they look like they're not doing anything). Often, they can be killed in such a way that both ends of the connection are unaware.

I first learnt this lesson when moving an application from a `docker-compose` based development to a `docker swarm` based test environment. When developing on a single machine, I was using the default `bridge` network, and all was well with the world. As soon as a more complicated network driver was introduced (in this case an `overlay` network, which creates a logical network across multiple swarm nodes), I started to see connectivity issues.

With a database connection, I might see an error when trying to run a query. That's not ideal, but at least there's an error you can catch and deal with.

Another critical component of this application is a message queue. The thing about the way this application uses message queues, is that the application just sits there waiting for a message to arrive, before processing it and sending it on. If the underlying TCP connection gets killed, the application just doesn't receive any messages.

So how do you resolve this in the docker world?

I don't think you can ever by 100% confident that a TCP connection will not get killed, and therefore you should assume that it *will* get killed. The best you can do is to quickly detect the loss of connectivity, and recover. Some form of heart-beat is probably the best way to do that, ie receiving a message from the other end of the connection. Not receiving a message (aka "pulse" or "heart-beat") in the expected time-frame is an indication that something may be wrong, and your application is quickly running out of oxygen. I believe that in the docker world, the best thing to do at this point is to log a high-priority message (eg `critical`) and cause the container to terminate with an error code. Docker will then take care of spinning up a replacement container, which should reconnect and take over.

If the heart-beat concept is already implemented in the libraries you are using, it's usually pretty trivial to implement, but a home-grown solution using timers is also fairly easy to develop (and I have a couple of examples, which is a subject of another post).

For the technology stack I am using, heart-beating has been implemented like this:
*Oracle database connection
  - home-grown, issue a simple query periodically
* STOMP message queue connection from PHP
  - used [capablue/stomp](https://github.com/capablue/stomp), a fork of [reactphp/stomp](https://github.com/reactphp/stomp) which offers heart-beating
* STOMP message queue connection from node.js
  - home-grown, periodically send a message to a topic, subscribe to that topic to receive the response
