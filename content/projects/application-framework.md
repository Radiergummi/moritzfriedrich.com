---
title: "Application Framework"
date: 2020-03-17T10:13:38+01:00
draft: false
tags:
    - project
    - messengerpeople
---

**Driven by the need to reuse domain-specific code, I started working on a unified base framework for every application we run. At the time of writing, this amounts 
to 37 individual services, both console and web applications.**  
Before the lamentation starts, let me be clear on one important point: I did _not_ build a fresh framework, of course! There's lots of smarter people than me who
have put a lot of thought into existing options. Nevertheless, there's always stuff we layer on top, services we regularly interact with or middleware that all
applications need. Extracting all of these requirements into modules and making them available on our internal composer registry was the first step, quickly 
followed by pulling all of those modules into a single package required by all applications.  
By now, the framework has grown pretty mature and includes the following things, listed in no particular order:

 - [A MySQL client](#a-mysql-client)
 - [A RabbitMQ abstraction layer](#a-rabbitmq-abstraction-layer)
 - [An HTTP client](#an-http-client)
 - [An application base-class](#an-application-base-class)
 - [A wrapper for Slim](#a-wrapper-for-slim)
 - [A wrapper for Symfony Console](#a-wrapper-for-symfony-console)
 - [A manager for threaded](#a-manager-for-threaded-workers)
 - [A template engine](#a-template-engine)
 - [A caching system](#a-caching-system)
 - [A debug analyzer](#a-debug-analyzer)
 - [Automated error reporting](#automated-error-reporting)
 - [Lots of common middleware](#lots-of-common-middleware)
 - [Other helpers](#other-helpers)

A MySQL client
--------------
Working on a greenfield project, I'd choose Doctrine, Eloquent or Propel any day. As is often the case, though, software tends to exist and 
have some kind of sub-par SQL handling that does lots of things, but almost never use a proper ORM. When I started, raw SQL queries with manual escaping was 
sprinkled across millions of lines of code! This situation was obviously unbearable, from both security and usability perspectives. I settled out to create a new
client that relies on prepared statements using `PDO` instead of `mysqli`, includes an (optional) query builder and handle errors properly.  
```php
<?php # Opening tag required due to https://github.com/alecthomas/chroma/issues/210

// Translates to:
// SELECT `t`.`uuid`, `u`.*, `t`.`name` FROM `users` AS `u` 
//   INNER JOIN `teams` AS `t` ON `u`.`team_id` = `t`.`id`
//   WHERE `u`.`created_at` >= ?
//   LIMIT ?
$query = Query::table('users')
    ->select(['teams.uuid', 'users.*', 'teams.name'])
    ->where('users.created_at', '>=', new DateTime($since))
    ->innerJoin('teams')
    ->limit(500);

// Retrieves the first 500 users with their team data joined, grouped by their team UUID:
// [ 'ebc39be3-...' => [ <User 1>, <User n> ], '6a09f7c9-...' => [ ... ] ]
try {
    $groupedRows = $database->query($query)->grouped();
} catch (Framework\Database\Exeptions\ConnectionException $exception) {
    // ...
} catch (Framework\Database\Exeptions\QueryException $exception) {
    // ...
}
```
This shows several cool details of the library: The query builder supports named table references and aliases, automatic joins, column and table quoting as well as
free ordering of the query parts. As I write the query builder in my spare time, it's by no means complete--it doesn't support nested queries, unions, conditions or
more complex joins. Still, this covers roughly three quarters of all queries we need, so it's a huge step up from the existing code.  
Additionally, there's one more cool thing to mention here: The `Database::query()` and `Database::execute()` methods allow passing both a string query or a `Query` 
instance as the first parameter, with any bindings as the second one. As you can see from the example above, we don't use question marks in the query builder 
methods but insert the values directly--but the call to `query()` does not mention them!  
This works because the database library checks for this under the hood:
```php
<?php # Opening tag required due to https://github.com/alecthomas/chroma/issues/210
public function query($query, ?array $bindings = null): ?ResultInterface
{
    if ($query instanceof QueryInterface) {
        $bindings = $query->getBindings();
    }

    return $this->runQuery($query, $bindings);
}
```

As our code relies on complex queries and existing schemas, it doesn't translate into ORM logic all that well; therefore, the combination of a mostly-complete query
builder, and a gracefully progressive PDO wrapper are currently the best compromise.

A RabbitMQ abstraction layer
----------------------------
As [RabbitMQ forms the backbone]({{< ref "rabbitmq.md" >}}) of our messaging infrastructure, we have a strong requirement for
the [php-amqplib library](https://github.com/php-amqplib/php-amqplib), which is the official RabbitMQ client for PHP. It leaves a lot to be desired, though, as
it mostly just exposes the AMQP API to PHP, has broken `@throws` annotations and has both confusing method names and error messages.  
Our message queue client simplifies the API, provides a simplified exception system (3 instead of over 21!), transparently serializes payloads and headers and 
takes care of automatic connection restoration in case of a network split.

An HTTP client
--------------
While there's Guzzle, I hate it's completely undiscoverable API: All options must be passed in an array with string keys. Every single time
I need to add an option, I have to take a peek into their documentation. While they have cool stuff like promises, streaming or parallel requests, our HTTP 
client does lots of things better, has native [PSR-7](https://www.php-fig.org/psr/psr-7/) support and features an object-oriented API with self-explanatory 
methods. Take a look at the following example:  
```php
<?php # Opening tag required due to https://github.com/alecthomas/chroma/issues/210
$responseBody = Framework\Http\Client::post($url)
    ->withAuthorization(HttpRequest::AUTHORIZATION_BEARER, $token)
    ->withHeader('Foo-Bar', 'Baz')
    ->withBody([
        'automatically_serialized' => true
    ])
    ->asJson()
    ->run()
    ->getParsedBody();
```
The client is highly flexible, provides convenience methods for any operation on the request or response and works with cloned instances exclusively. Under the 
hood, we use the PHP stream context to perform the actual requests, so it performs pretty well, too. Whatever other features Guzzle has, this is more than 
sufficient for our use-cases.

An application base-class
-------------------------
When I started working on the first framework version, there was neither composer support nor any auto-loading: Every single PHP file contained `require_once` 
statements at the start of the file, and it was _a mess_. As I started to modularize the code, the first thing I did was add composer to the mix, which provided 
free auto-loading and module installation. The next-most important thing was getting _some_ kind of dependency injection working: Having to create lots of instances
across the code or rely on global variables was definitely not the way to go forward.  
At this point, we agreed on [Pimple](https://pimple.symfony.com/). The way our applications were structured at that point, however, meant using the container as a 
[service locator](https://en.wikipedia.org/wiki/Service_locator_pattern), which is a non-optimal solution to dependency injection: All code is tightly coupled to 
the container and the injected classes.

Therefore, I soon decided to switch to [PHP-DI](https://php-di.org/) instead, bringing all kinds of goodness like auto-wired dependencies, cached definitions and
interface-only coding! This might not seem to revolutionary if you're used to Symfony applications, but keep in mind that we switched from plain old `require` to 
modern dependency injection in well under a year. 

To keep application bootstrapping uniform across all projects, the base class performs all required steps in its constructor: That includes configuration loading,
`.env` inclusion (in development mode), container building and dependency resolution, error handler setup and so on---everything you'd usually do in the 
`index.php`.

A wrapper for Slim
------------------
Searching for a web framework to use as a foundation for our web apps, I finally settled on [Slim](https://www.slimframework.com/). While Symfony or Laravel are 
certainly more complete, mature, and feature-rich, they are extremely opinionated about the way to do things. Migrating existing code to them is almost impossible;
rewriting millions of lines of code is unfeasible for a startup. That ruled out practically all "big" names, leaving us with the last option on the list: Slim. It 
imposes almost no restrictions, making for a blank slate to get started with---exactly what we needed!  
Everything Slim does is handling HTTP requests and responses, and keeping in line with Unix philosophy, it does that extremely well. Since version 4, however, I 
suspect a bunch of enterprises switched started using it and bloat started to roll in: Suddenly, the 
[recommended `index.php`](https://github.com/slimphp/Slim-Skeleton/blob/8d8916453017ec9c3b7161b4d71536327484356e/public/index.php) file requires heaps of generic
class declarations and setup work, having a JSON response helper was moved into a separate decorator module and explicitly declaring response factories was deemed
a good idea somewhere along the path. Most of these changes originated in the new ability to choose a custom PSR-7 implementation, which is somewhat nice, but 
mostly _absolutely pointless_. I respect the freedom Slim tries to give me, but searching the auto-loader tree for existing implementations of a completely generic
interface was too much senseless overhead.  
Finally, I decided to re-write several of Slim's core classes, make our own PSR implementation (with the decorated behaviour built-in) and stuff it all into the
framework. Contrary to my apprehensions, this has actually improved performance and made the code _way_ easier to reason about. The following is everything required
for application bootstrap currently:
```php
<?php # Opening tag required due to https://github.com/alecthomas/chroma/issues/210
define('ROOT', __DIR__ . DIRECTORY_SEPARATOR . '..' . DIRECTORY_SEPARATOR);
define('CONFIG', ROOT . 'config' . DIRECTORY_SEPARATOR);

require ROOT . 'vendor/autoload.php';

$application = new Framework\Application(CONFIG);

require_once CONFIG . 'routes.php';
require_once CONFIG . 'middleware.php';

$kernel = $application->make(Framework\Http\Kernel::class);
$response = $kernel->handle(
    $request = Framework\Http\RequestFactory::createFromGlobals()
);

$kernel->send($response);
$kernel->terminate($request, $response);
```

As the framework contains a [Facade implementation](https://refactoring.guru/design-patterns/facade/php/example), all routing happens via the facade implementation:
```php
<?php # Opening tag required due to https://github.com/alecthomas/chroma/issues/210
Router::post('/path/to/endpoint', [MyController::class, 'handlerMethod']);
```

Due to a custom dependency resolver strategy, all controller classes have full dependency injection support. Controller methods have access to application 
dependencies, request and response instances as well as URI arguments and query/body parameters, in any order. This is on-par with Symfony or Laravel, while being 
cheaper in terms of performance budget!

A wrapper for Symfony Console
-----------------------------
The Slim components are awesome, but it kind of sucked console applications, while also having a need for dependency injection, worked completely different to web
apps. After a bit of tinkering, I refactored the `Application` class to support different kernels: One for the web, another for the console. There are other use
cases I'm thinking about where a custom kernel could prove useful.  
This allows us to run console applications that have native dependency injection on all command classes!

A manager for threaded workers
------------------------------
As I wrote in the [massively parallel PHP project]({{< ref "massively-parallel-php.md" >}}), we run lots of queue workers that must execute lots of worker threads 
in parallel to scale properly. This required the most painful-to-build component in the framework, the one causing me the most headache. As we started considering 
PHP threads, we started out with the only option back then---[`pthreads`](https://www.php.net/manual/en/intro.pthreads.php). As 
[the author of `pthreads` himself writes](https://blog.krakjoe.ninja/2019/02/parallel-php-next-chapter.html), it contains lots of bugs, is hard to understand and
debug and keeps crashing. Let me quote my favorite passage from that article:
> I spent many hundreds, possibly thousands of hours writing and rewriting pthreads until it is what you see today, a kind of monster that about 4 people really 
> understand excluding myself, that only the same number of projects have really managed to deploy with any success.  
> --Joe Watkins aka. Krakjoe, author of pthreads

While I'm quite happy to report that we got `pthreads` to work _almost_ without any bigger issues, it's deprecated by now and won't work properly with newer PHP
versions. Luckily, a new alternative is available: [The `parallel` extension](https://www.php.net/manual/en/intro.parallel.php)! Parallel has a new threading model
built-in and relies on message passing, a little similar to the way [goroutines](https://golangbot.com/goroutines/) work. Even better, the way it is structured 
allowed me to build a lean, elegant wrapper on top. The wrapper exposes a `Task` as the highest-level abstraction layer---Tasks have full support for dependency 
injection and don't even have to know they run in their own thread, which makes them ideal for userland code.  
A task is always executed in a `TaskPool`, which spawns a number of workers and makes sure crashed workers are respawned immediately, keeping the pool at the 
desired size. This works from both CLI commands and standalone worker scripts, making it super-simple to delegate tasks into separate threads: Say, you want to 
process multiple files in parallel, or download something while keeping the CLI app busy with something else. All we have to do is spawn a task pool, tell it what 
task to work on, and take care of other things. It really feels like magic :)  
A fully-working worker script looks like the following:
```php
#!/usr/bin/env php
<?php
define('ROOT', __DIR__ . DIRECTORY_SEPARATOR . '..' . DIRECTORY_SEPARATOR);
define('CONFIG', ROOT . 'config' . DIRECTORY_SEPARATOR);

require ROOT . 'vendor/autoload.php';

$application = new Application(CONFIG);
$container = $application->getContainer();
$taskPool = new Framework\Multitasking\TaskPool([
    FirstTask::class => 3,
    SecondTask::class => 7
]);

$taskPool->bootstrap(16);
$taskPool->keepAlive();

exit(0);
```
This will spawn 16 workers to handle both tasks, then wait for them to finish. What do the numbers mean, though? The pool includes a weighted priority algorithm. 
At bootstrap time, the pool size (16 in this case) will be split according to the weights assigned to the tasks (3 and 7 here). Threads will be spawned according to
the ratio of the given weights---in this case, it would spawn 5 workers for the first task and 11 for the second.

A template engine
-----------------
The template engine is basically just an extended version of the [Slim PHP view library](https://github.com/slimphp/PHP-View), which is _almost_ what I wanted but 
lacked features such as proper partials or default contexts.

A caching system
----------------
The caching component is somewhat more elaborate: Throughout the framework, there are several constructs that allow access to key-value stores. All of them 
implement the `Store` contract, which defines a small set of methods (`get()`, `set()`, `has()`, `drop()` and so on) to access data in the store. Based on this, the
caching components allow using any store implementation as its backend; most prominently, this is a decorator for [PRedis](https://github.com/nrk/predis). The cache
classes, however, also implement [PSR-6](https://www.php-fig.org/psr/psr-6/) and [PSR-16](https://www.php-fig.org/psr/psr-16/)---they are fully compatible to the 
[PHP-Cache test suite](https://github.com/php-cache/integration-tests)! This is really neat because we can use our own cache implementation in any third-party 
library that accepts a PSR cache.

A debug analyzer
----------------
Debugging PHP errors can be a pain: Stack traces are hardly helpful and tracking state is unnerving. To make this easier, I built a drop-in error handler that can
be used in debug mode (but doesn't have to) that provides something similar to the ["Ignition" handler by Flare](https://flareapp.io/ignition) for Laravel: A custom
error page that shows details on the error, a user-friendly and syntax-highlighted stack trace that includes snippets from the source code, environment info and 
resolution steps for common problems. This was a fun challenge, actually!

Automated error reporting
-------------------------
With many applications running in Kubernetes, it becomes completely impossible to keep track of all application log files. Therefore, we have a monitoring queue set 
up that a message is sent to whenever an exception is thrown. These messages include the full exception trace, file origin, current environment details, request 
details (if we're running in a web app) and some other details. The queue is consumed by a small monitoring worker that analyzes the traces: Depending on the 
severity and a rule map, it sends notifications via Slack, mail and WhatsApp Business (for critical exceptions).  
It then resolves the code to the currently deployed revision and creates an entry in our monitoring database, which allows full-scale analysis over any exception 
ever thrown anywhere in the code! I can view a time-based graph for any error message there ever was since I turned that thing on, which is pretty neat :)

Lots of common middleware
-------------------------
Additionally, the framework sports lots of common middleware---authentication, CSRF tokens, OAuth-scope based authorization, IP address resolution, language and 
content type negotiation, automated API versioning and so on. This always makes use of existing third-party middleware where possible.

Other helpers
-------------
There are loads of other stuff included by now: Functions to use in templates, facades for common classes, wrappers for third-party APIs we use often... the whole 
thing has become quite powerful by now!
