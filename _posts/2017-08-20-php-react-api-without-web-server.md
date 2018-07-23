---
title: PHP-react API without a web server
---

I have been pretty impressed with the performance benefits I've seen when running a daemonized PHP process based on an event loop, as compared to the traditional MVC app running via a web server (eg, Apache). I think a lot of the performance gains come from avoiding per-connection startup, and connecting to resources such as a database.
Whilst profiling PHP application performance in the past, the fastest I've seen a database connection be established is ~50ms (with a connection pool bringing this down to the 5-10ms range).
The react/stomp configuration I've previously blogged about, when used with prepared database queries, will consistently process requests within 1.2 to 1.8ms. This is against a production database (the same one that it takes 50ms to establish a connection to), and running 3-4 database queries in the mix.

Based on that, I want to explore whether a similar configuration is able to serve a REST API (with react waiting on incoming HTTP requests rather than messages).

I played with a couple of PSR-7 routers to achieve this. I liked `league/route` (which is based on `nikic/fastroute`), but it was only able to route a single request.
`aura/router`, however, was up to the job, so this is what I have so far:

```json
{
  "require": {
    "react/http": "0.*",
    "mkraemer/react-pcntl": "~2",
    "aura/router": "*",
    "zendframework/zend-db": "*"
  }
}
```
And the code itself:

```php
<?php
require 'vendor/autoload.php';
use Psr\Http\Message\ServerRequestInterface;
use Psr\Http\Message\ResponseInterface;
use React\EventLoop\Factory;
use React\Http\Response;
use React\Http\Server;
$db = new \Zend\Db\Adapter\Adapter([
    'driver' => 'pdo_sqlite',
    'database' => '/srv/app/data/sqlite.db',
]);
$stmt = $db->createStatement('select sqlite_version() as version');

$loop = Factory::create();
$pcntl = new MKraemer\ReactPCNTL\PCNTL($loop);
$pcntl->on(SIGTERM, function () {
    echo 'Bye'.PHP_EOL;
    die();
});

$counter = 0;
$routerContainer = new \Aura\Router\RouterContainer();
$map = $routerContainer->getMap();
$map->get('version', '/version', function(ServerRequestInterface $request, ResponseInterface $response) use (&$counter, $stmt){
    $result = $stmt->execute();
    $response->getBody()->write(json_encode([
        'version' => 1,
        'db' => $result->current()['version'],
        'count' => ++$counter]));
    return $response;
});
$matcher = $routerContainer->getMatcher();
$count = 0;

$server = new Server(function (ServerRequestInterface $request) use ($matcher, &$count) {
    $response = new Response();
    //route all of the things
    try {
        $route = $matcher->match($request);
        if (!$route) {
            return $response->withStatus(404);
        }
        $handler = $route->handler;
        $response = $handler($request, $response);
        echo sprintf('[%s] [%d] %s %s%s', $response->getStatusCode(), ++$count, $request->getMethod(), $request->getUri(), PHP_EOL);
        return $response;
    } catch (\Throwable $e) {
        var_dump($e);
    }
});

$socket = new \React\Socket\Server('0.0.0.0:'.getenv('API_PORT'), $loop);
$server->listen($socket);
echo sprintf('Listening on %s%s', str_replace('tcp:', 'http:', $socket->getAddress()), PHP_EOL);
$loop->run();
```

I hit this this with 1000 requests to see if it could keep up:

```bash
ab -n 1000 -c 10 -l http://localhost:8000/version
```

What if the database connection is lost?
With a message queue setup, I would just have the container die in the knowledge that docker would restart the service, and due to the message queue no messages would be lost. Perhaps this needs a message queue in the mix? Or maybe it should stay up, return 500 responses, but keep trying to connect to the database. That should be easy enough to achieve with an event loop...
