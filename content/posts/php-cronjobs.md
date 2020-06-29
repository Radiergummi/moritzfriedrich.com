---
title: "Simple Cronjobs for CLI apps using systemd"
date: 2020-01-16T13:29:13+01:00
showDate: true
draft: false
tags: 
    - blog
    - story
---

Most applications will need some kind of asynchronous processing happening in the background: Calculating statistics, sending emails or processing large files. These tasks are probably abstracted into some kind of _job queue_, ready to be fetched by a worker.  
Now, whenever you want to perform a task in regular intervals (say: Calculating monthly billing positions and sending them to Stripe for invoicing), you need some kind of scheduler. The most straightforward and probably most common solution is to simply create a _cron job_. While cron jobs might work for the most part, they bring lots of hidden complexity:

 - What happens if an execution takes longer than expected and ends up still running when the next scheduled execution starts?
 - Where do logs go? Do you even log your cron job results? _All_ of them?
 - How do you randomize execution times to prevent hitting a third-party API rate limit, for example?
 - Can you limit system resource usage for them?

The solution to these and several other problems are [systemd's timers](http://man.he.net/?topic=systemd+timer&section=all). I know hating on systemd is still fashionable as always, but whatever your opinion is, it's not going to go anywhere anytime soon, and timers are awesome!

Setting up a timer
------------------
To get systemd to repeatedly execute something, you always need two parts:

 1. **A service unit.** This can really be just an ordinary service unit, but for our use case, we use a minimal service configuration only. This "service" is the command being executed for your job - for the sake of this post, let's assume `/opt/job do-work`.
 2. **A timer unit.** As with all things systemd, timer units also live in your systemd directory (probably `/etc/systemd/system/`) and can (and must) be separately enabled. The timer is configured to execute the service if a set of conditions match.

I don't know about your application, but mine have lots of jobs to do. Hearing this, I immediately thought how awful it'd be to create **two** configuration files and type a bunch of `systemctl` commands, just to achieve the same effect as putting a single line of text in my crontab.  
But bear with me for a moment: I promise there's a more elegant solution ([skip to the TL;DR](#wildcard-services)).

Writing the service
-------------------
Say we're trying to execute the previously mentioned `/opt/job do-work`. The following service configuration would achieve this task:

```ini
[Unit]
Description=job worker
After=network.target

[Service]
ExecStart=/opt/job do-work 

[Install]
WantedBy=multi-user.target
```

This snippet is pretty simple - from top to bottom:

 - The `Description` in the unit section will be shown in the log files, so this is mostly cosmetic.
 - The `After` denotes that our job worker requires network access to run, so it can't be started before the network daemon is initialized.
 - The `ExecStart` contains the command we intend to execute on starting the service.
 - The last line, `WantedBy`, is the systemd way of saying our service requires at least runlevel 3. This is the level just below initialization of the GUI systems, so it basically means _"this service requires the system to be up"_.

We save that service unit file as `/etc/systemd/system/job.service` for now. After executing `systemctl daemon-reload`, you should be able to run `systemctl start job` and review the log output using `journalctl -u job`.

Writing the timer
-----------------
Now that we have a working service, we can create the timer unit.

> As an aside, I think this is much more in line with the Unix philosophy (_"Do one thing, and do it well"_) than any cron approach: The job to execute and the definition of the execution schedule are two separate things, decoupling potentially complex scheduling from the act of doing a thing.

Further assume we want to execute our command every 60 minutes, or rather: With a pause of 60 minutes between individual executions. Additionally, we'd like to add a random delay of 0&ndash;30 seconds.  
The timer unit could look like so:

```ini
[Unit]
Description=job timer

[Timer]
Unit=job.service
OnUnitInactiveSec=60m
RandomizedDelaySec=30
AccuracySec=1s

[Install]
WantedBy=multi-user.target
```

Now the timer section is the most interesting here:

 - The `Unit` defines the service this timer will execute. It expects the name of our service unit file.
 - The second directive, `OnUnitInactiveSec`, is one of several possible timer settings. It accepts a time interval, basically the time to wait before starting the service again, _counted from the end of the previous execution_.
 - `RandomizedDelaySec` instead accepts a number of seconds that a random interval will be chosen from. The execution will then be delayed by that random interval.
 - The last directive `AccuracySec` defines the accuracy the timer will be checked, so a lower value means the timer will fire more accurately. The minimum value is `1us`, but we probably don't need this much precision in our case.

The timer file should be saved as `/etc/systemd/system/job-work.timer`. As with the service unit, after executing `systemctl daemon-reload`, you should be able to run `systemctl start job-work.timer`. Don't forget to enable the timer using `systemctl enable job-work.timer`.

To monitor when your job is going to be executed the next time, you can use `systemctl list-timers`, which will list the all timers with their last and next execution dates, the time left until the next and passed since the last one as well as the service unit being executed. A marvelous command!

After having performed the above step, we should have a running configuration, with our `/opt/job do-work` command being run approximately every hour. Phew. While you can obviously optimize scaffolding the timer setup, repeating this for _every single job type_ sounds like way too much work.  
So as promised, there's a better solution than lots of separate configuration files!

Wildcard services
-----------------
There exists a neat little feature in systemd to define parameterized services. If the service unit file contains an `@` character in its file name, just before the `.service`, it will be treated as a "template unit file". You can then refer to the service with any characters after the `@` being treated as a dynamic parameter, available inside the unit file as `%i`. That sounds pretty confusing but is easy in practice, so let's transform our above setup to use the instance parameter.

Creating a wildcard unit
------------------------
To accept a parameter, we need to change the filename of our service unit to the following:

```
/etc/systemd/system/job@.service
```

Inside the unit, we may now use the `%i` placeholder, which will be replaced during execution:

```ini
[Unit]
Description=job worker for %i
After=network.target

[Service]
ExecStart=/opt/job %i 

[Install]
WantedBy=multi-user.target
```

If you scroll back up to the original service unit, all we did was replace the `do-work` subcommand with our `%i` placeholder! Now (after the usual `systemctl daemon-reload`, that is), we can start our service with the placeholder being passed dynamically on the command line!  
To resume execution of our `do-work` command, we can call the service like so:

```sh
systemctl start job@do-work.service
```

Behind the scenes, this will trigger the `ExecStart` command line and replace the `%i` with `do-work`. And now that we got this working, we should update our timer unit:

```ini
[Unit]
Description=job timer

[Timer]
Unit=job@do-work.service
OnUnitInactiveSec=60m
RandomizedDelaySec=30
AccuracySec=1s

[Install]
WantedBy=multi-user.target
```

Now, we can simply copy-paste the timer unit and insert our desired sub-command instead of `do-work` in the `Unit` section.  
But wait, there's more!

Creating a wildcard timer
-------------------------
Why stop there? We can of course also use the wildcard placeholder inside our timer!

```ini
[Unit]
Description=job timer %i

[Timer]
Unit=job@%i.service
OnUnitInactiveSec=60m
RandomizedDelaySec=30
AccuracySec=1s

[Install]
WantedBy=multi-user.target
```

Here, we replaced the sub-command with `%i` again, allowing us to use this timer dynamically for any job that should run hourly. So after saving it as `/etc/systemd/system/job-hourly@.timer` (yes, you guessed right: `systemctl daemon-reload`), we can finally start adding commands using systemctl only:

```sh
systemctl enable job-hourly@first-command.timer
systemctl enable job-hourly@second-command.timer
systemctl enable job-hourly@third-command.timer
```

If you need other schedules too, simply copy-paste the wildcard unit, change the timer options and enable the timers for your desired parameter.

Finally, we have a cron-less, flexible and best practice conformant scheduler system that can be infinitely expanded and is portable. Awesome!










