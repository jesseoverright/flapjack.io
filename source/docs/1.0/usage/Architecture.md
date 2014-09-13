## Contents

- [Architecture](#architecture)
- [Components](#components)
- [Resque](#resque)
- [Redis Database Instances](#redis_database_instances)


<a id="architecture">&nbsp;</a>
## Architecture

![Flapjack architecture diagram as of 2013-09-27](/images/architecture.png)

```
Check Receivers ---> Executive ---> Gateways
```

**Check Receivers** read check data generated by external programs; Flapjack ships with a Nagios checks receiver, and connectors to other sources of check data could also be written. Checks define the entity they refer to as being in one of three states (`OK`, `WARNING` or `CRITICAL`), are timestamped and can optionally include further data about the reason for the current entity state. The checks are added onto an events queue by the check receivers.

The Flapjack **Executive** reads events from the events queue and determines which contacts should be notified for them. It runs filters to prevent notifications being fired too often for the same check.

Flapjack **Gateways** serve as the public interface to Flapjack. There are *unidirectional* notifiers (e.g. `email` and `sms`, which read notifications from a queue and send them out to registered contacts, there are *bidirectional* notifiers (e.g. `jabber`, `pagerduty`) which do the above and also offer a back-channel for acknowledgements etc., and there are also `web` and `api` gateways for retrieving reporting data, creating maintenance periods, acknowledging checks, etc.

### Processor and Notifier

In the beginning, there was Executive. This has been split into two separate components, *processor* and *notifier*.

Flapjack processor processes events from the *events* list in redis. It does a blocking read on redis so new events are picked off the events list and processed as soon as they created.

When executive decides somebody ought to be notified (for a problem, recovery, or acknowledgement or what-have-you) it generates a job on the *notifications* queue.

Notifier picks up jobs from the *notifications* queue, looks up contact information, applies notification rules and per-media intervals and rollup logic, and then creates notification jobs for the appropriate notification gateways.

<a id="components">&nbsp;</a>
## Components

Executables:

  * `flapjack` => starts multiple components ('pikelets') within the one ruby process as specified in the configuration file.
  * `flapjack receiver nagios` => reads nagios check output on standard input and places them on the events queue in redis as JSON blobs.
  * `flapjack receiver nsca` => reads the nagios' commandfile output and places them on the events queue in redis as JSON blobs.
  * `flapjack flapper` => runs a daemon that intermittently listens on port 12345 (one minute on, one minute off, ...)
    to be used for generating heartbeat events for end to end monitoring of flapjack
  * `flapjack import` => creates contacts and entities in redis, reading from JSON formatted data files
  * `flapjack simulate fail` => simulates a failed check by creating a stream of events for flapjack to process

Usage information for each executable is provided below.

Pikelets:

*   `processor` => processes monitoring events off the *events* queue (a redis list) and decides what actions to take (generate notification event, record state changes, etc)
*   `notifier` => processes notification events off the *notifications* queue (a redis list) and works out who to notify, and on which media, and with what kind of notification message. It then creates jobs for the various notification gateways (below)

*   notification gateways
    * `email` => generates email notifications (resque, mail)
    * `sms` => generates sms notifications (resque)
    * `jabber` => connects to an XMPP (jabber) server, sends notifications (to rooms and individuals), handles acknowledgements from jabber users and other commands (blather)
    * `pagerduty` => sends notifications to and accepts acknowledgements from [PagerDuty](http://www.pagerduty.com/) (NB: contacts will need to have a registered PagerDuty account to use this)

*   other gateways
    * `web` => browsable web interface (sinatra, thin)
    * `jsonapi` => HTTP API server (sinatra, thin)
    * `oobetet` => "out-of-band" end-to-end testing, used for monitoring other instances of flapjack to ensure that they are running correctly

Pikelets are flapjack components which can be run within the same ruby process, or as separate processes.

The simplest configuration will have one `flapjack` process running processor, notifier, web, and some notification gateways.  The same process will also run the receiver, which receives events from Nagios and places them on the events queue for processing.

<a id="resque">&nbsp;</a>
## Resque

### Resque Workers

We're using [Resque](https://github.com/resque/resque) to queue email and sms notifications generated by flapjack-executive. The queues are named as follows:
- email_notifications
- sms_notifications

Note that using the flapjack_config.yaml file you can have flapjack start the resque workers in-process. Or you can run them standalone with the `rake resque:work` command as follows.

One resque worker that processes both queues (but prioritises SMS above email) can be started as follows:

```text
QUEUE=sms_notifications,email_notifications VERBOSE=1 be rake resque:work
```

resque sets the command name so grep'ing ps output for `rake` or `ruby` will NOT find resque processes. Search instead for `resque`. (and remember the 'q').

To background it you can add `BACKGROUND=yes`. Useful documentation is available in [Resque's README](https://github.com/resque/resque/blob/master/README.md)

### Resque Queue Management with resque-web

If you need to inspect or purge the queues managed by resque you'll want to start up an instance of resque-web. This will by default connect to redis database 0 which is fine for production but in development you'll need to specify database id 13 (or whatever you have it set to in the flapjack config) e.g.:

```bash
resque-web -p 4082 -r localhost:6379:13
```
This will start a resque-web instance listening on port 4082, connecting to the redis server on localhost port 6379, and selecting database 13.

<a id="redis_database_instances">&nbsp;</a>
## Redis Database Instances

We use the following redis database numbers by convention:

* 0 => production
* 6 => quickstart
* 13 => development
* 14 => testing