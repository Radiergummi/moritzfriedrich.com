---
title: "Massively Parallel PHP"
date: 2020-06-29T12:35:04+02:00
draft: false
tags:
    - project
    - messengerpeople
---

With most large software systems, sooner or later a need arises to perform certain jobs in parallel. We usually solve this problem with some sort of job queue which
schedules work to be processed later, asynchronously, in a separate process. While PHP might not be the first language that jumps to mind for long-running workers,
it was still the best possible choice at MessengerPeople. How so? We have a huge ecosystem of applications, shared libraries and conventions, all built with PHP. 
Being able to reuse those was an enormous benefit during development. Additionally, no matter how perfect another language may be--if your engineers don't know it,
they (including you) are bound to fail.  
This lead me to creating a simple system to execute PHP code in parallel. Analyzing the situation, the wishlist included the following:
 - Be able to spawn multiple worker threads per instance
 - Have worker code be oblivious of being run in a worker
 - Rely on existing code and conventions as much as possible

At the time of building the foundation for our workers, the only way to get PHP to spawn userland threads was using 
[the `pthreads` extension](https://blog.krakjoe.ninja/2019/02/parallel-php-next-chapter.html), which is neither safe, nor elegant, nor even easily understandable, 
as even its author has written. After lots of nights spent furiously punching my keyboard, I finally managed to bend pthreads to my will; it wasn't pretty, but the
code got the job done. Basically, I moved all the specifics of the extension behind the curtains, and exposed a much clearer API to our applications, consisting of 
tasks, workers and pools. A task is something to be executed, a worker executes such a task, and a pool consists of multiple workers executing one or more tasks.  
All of a sudden, user-userland code is surprisingly simple:
```php
<?php
class MyTask extends Framework\Support\MultiTasking\Task
{
    public function run(): void {
        // do stuff
    }
}
```

The task would be run in a try-catch block; if it threw an exception, it would be considered failed and restarted. If no exception was thrown, the worker shut down
cleanly, as its job was obviously done. Worker pools could be configured to use a fixed number (or proportional fraction of available) workers, and automatically 
spawn replacements for any stopped workers. Behind the scenes, we would spawn a new thread for every worker and bootstrap it using the main framework entry point,
which would take care of initializing the autoloader, configuration loading and dependency injection setup. At the time the task's `run()` method would be called,
everything was ready and initialized, providing a clean and modularized runtime environment to every single thread.

This is definitely not the most elegant or efficient approach (every single worker keeps the same, large, runtime in memory!), but it avoids typical threading 
pitfalls with shared memory and effectively prevents memory leaks PHP is pretty vulnerable to, as every reference from tasks are cleaned up on completion. Plus, and
most importantly, _the task implementation is so simple, anyone on the team could add new tasks to our worker application_.

With the "front-end" part done, we can boot up an application with a pool of, say, 32 worker threads, and let it handle a task that listens to our job queue. Every 
worker thread listens to the RabbitMQ server for new jobs; if it crashed during handling, the message went back into the queue. As we, unsurprisingly, run all of 
this within Kubernetes of course, the application runs in a Docker container which is spread out among a dynamically resized number of pods.  
Putting all of this together, we end up with a distributed, auto-healing, dynamically resizable, completely horizontally scalable, worker system written entirely in
PHP.

-----

Now the story could end here, but of course the world doesn't. So as it goes, Joe Watkins (the developer of the pthreads extension) figured out there might be a better approach to threading in PHP. Thus, he started
