---
title: ActiveMQ connections vs ephemeral containers
---

When running applications in a containerised environment, you cannot assume that any given container will always be there. A container might die, an update might cause it to be restarted on another node, or the [chaos monkey](https://github.com/Netflix/SimianArmy/wiki/Chaos-Monkey) might come along and randomly terminate it.

Here's a minimal PHP application which runs a react event loop and sends messages to itself each second. Every 2 seconds, it emits a message saying whether it thinks its connection is still good (this is pretty much the [example from react/stomp](https://github.com/reactphp/stomp#example):
```php
<?php
require_once('vendor/autoload.php');
$loop = React\EventLoop\Factory::create();
$factory = new React\Stomp\Factory($loop);
$client = $factory->createClient(array('host' => 'mq', 'port' => 61613));

$client
  ->connect()
  ->then(function ($client) use ($loop) {
    $client->subscribe('/topic/foo', function ($frame) {
      echo "Message received: {$frame->body}\n";
    });

  $loop->addPeriodicTimer(1, function () use ($client) {
    $client->send('/topic/foo', 'le message');
  });
});

$loop->addPeriodicTimer(2, function() use ($client){
  if ($client->isConnected()) {
    echo 'I am still connected'.PHP_EOL;
  } else {
    echo 'I am not connected'.PHP_EOL;
  }
});
$loop->run();
```

This is what happens when you kill the message server whilst a react/stomp loop is reading and writing from/to it. As you can see, the messages stop, but there is no sign of error, the stomp client still believes itself to be connected, and the code keeps happily running. It's just not doing anything useful.

```
mq_1   | 2017-02-22 09:46:19,487 WARN received SIGTERM indicating exit request
mq_1   | 2017-02-22 09:46:20,141 INFO waiting for cron, activemq to die
php_1  | I am still connected
php_1  | Message received: le message
php_1  | Message received: le message
mq_1   | 2017-02-22 09:46:21,839 INFO stopped: activemq (terminated by SIGTERM)
mq_1   | 2017-02-22 09:46:21,873 INFO stopped: cron (terminated by SIGTERM)
php_1  | I am still connected
php_1  | Message received: le message
php_1  | Message received: le message
php_1  | I am still connected
php_1  | Message received: le message
php_1  | Message received: le message
php_1  | I am still connected
mqtest_mq_1 exited with code 0
php_1  | I am still connected
php_1  | I am still connected
php_1  | I am still connected
php_1  | I am still connected
```

For use in a real containerised production environment (eg in docker swarm), our client should error or emit an event so that we can take some remedial action. That action may be to:

* try to reconnect
* die
* fail its health checks (docker will kill the container after a configurable number of health checks have failed), and restart it
* a combination of the above

Looking through the react/stomp issue list on github, I came across [https://github.com/reactphp/stomp/pull/38](https://github.com/reactphp/stomp/pull/38). I suspect that knowing when the underlying TCP connection dies is a great start, so I forked and applied the pull request myself.

So let's kick the tyres. Here's the updated composer.json:
```JSON
{
    "require": {
        "react/stomp": "0.2.1"
    },
    "repositories": [
        {
            "type": "vcs",
            "url":  "https://github.com/Deakin/stomp"
        }
    ]
}
```
Re-running the php container, then killing the message queue, results in:
```
$ docker-compose logs -f --tail=0
...
php_1  | I am still connected
php_1  | Message received: le message
php_1  | Message received: le message
php_1  | I am not connected
mqtest_mq_1 exited with code 0
php_1  | I am not connected
```
That's an excellent start. Now, let's add an event handler:

```php
$client->on('close', function(){
    echo 'My connection was closed!'.PHP_EOL;
    exit(1);
});
```
...and re-run:
```
$ docker-compose logs -f --tail=0
...
php_1  | I am still connected
php_1  | Message received: le message
mq_1   | 2017-02-22 11:36:23,959 WARN received SIGTERM indicating exit request
mq_1   | 2017-02-22 11:36:23,995 INFO waiting for cron, activemq to die
php_1  | Message received: le message
mq_1   | 2017-02-22 11:36:25,035 INFO stopped: activemq (terminated by SIGTERM)
mq_1   | 2017-02-22 11:36:25,036 INFO stopped: cron (terminated by SIGTERM)
php_1  | My connection was closed!
mqtest_mq_1 exited with code 0
mqtest_php_1 exited with code 1
```
Now that the container exits with a non-zero code, docker is aware that something is wrong. It can restart the container. Hopefully, docker is also aware that there is something wrong with the message queue container, and it also gets restarted.

At least as an administrator, you are more likely to notice a dead container than a live one that's not doing any useful work.
