---
title: parseable docker logs
---

I've spent most of the past year working on an [exciting new project](https://www.itnews.com.au/news/meet-genie-deakin-unis-virtual-assistant-for-students-453230) for my employer, an Australian university. I'm one of two senior programmer/architects of the system, and it was my idea to build the whole thing in docker, using message queues and daemonised workers to allow us to scale out to cater for future growth. The project is currently in a pilot release to ~500 students.

Once we started to build out all of our different workers (using docker-compose to spin up and link all of the containers required to have a full working environment on a developer machine), it seemed natural to create log messages which told the story of what each component was doing:

```
[router] got a message from brett (messageId=123456)
[router] about to process the message
[router] processing the message
[router] processed the message (in 5ms)
```
For a single developer and user, it is really easy to read and to tell what's going on, and it helped a lot during initial development.
Once it came time to spin up a testing environment with multiple users, it quickly became very hard to tell what was going on, because the logs no longer appeared as a consecutive series of related messages that told a story.

A docker best practice is that all logs should be written to `stdout` and `stderr`, which we've adhered to from day one, and that gives us the opportunity to fairly easily ingest those logs into a tool such as `Splunk` or an `ELK` stack (Elasticsearch+Logstash+Kibana). We happen to already have Splunk, and in my experience so far, it's been a fairly useful tool for aggregating and searching through various disparate log sources.

But since going to the effort to generate consistent, JSON-formatted log files from our docker containers, I'm falling more and more in love with Splunk as a method of finding out what the hell is going on in our system. Now, by default, all log messages contain a minimal set of keys which is sufficient to trace a single interaction from the user (who connects via a mobile app to a `websocket`) through various workers, RESTful web services, and finally back to the user).

A minimal JSON message looks like:

```json
{
  "service": "router",
  "messageId": "123456",
  "conversationId": "9876543",
  "user": "brett",
  "level": "info",
  "message": "something happened"
}

The other notable difference between the `JSON` message and the human-readable one, is that all bits of data (service, message id, the user, the processing time, the message itself (what happened)) are separated into their own keys. If you can be consistent about those key names across multiple services, then you can start to do things like find out which of your services are slow to process, which have more errors, which are more frequently used.

## Less is more, but more is also more ##
In the original story-based logs, it was reasonable to emit lots of messages. I've found that when trying to understand a single user interaction via Splunk logs, that it's actually better to have a single `info`-level message in the case of a successfully-processed message, or a single `error`-level message in case of failure. That means waiting to the last minute before logging anything, and including all relevant details in that one message. For example:

```json
{
  "service": "router",
  "messageId": "123456",
  "conversationId": "9876543",
  "user": "brett",
  "question": "show me my timetable",
  "intent": "timetable.show",
  "level": "info",
  "message": "routed message",
  "destination": "queue/timetable.show",
  "processing-time": "3ms"
}
```
Once these `JSON`-formatted logs have been ingested into Splunk (we currently use `fluentd`, but hope to use the splunk logger in future) then the full power of Splunk becomes apparent. All of our production and test logs are aggregated into separate Splunk indexes, and I can very quickly drill down into "show me all `error` events for the past hour", "show me all events for user X in the past 15 minutes".

![splunk screenshot]({{ site.baseurl }}/images/splunk-example-300x168.png "splunk screenshot")

I still think there is a place for story-based log messages, but I try to keep those at a `debug` level which is ignored by default in our test and production environments.
