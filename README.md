# Harrison

Harrison is an offline task system that uses Redis and Node.js to keep track of
tasks and push them out to ancillary worker processes. It is inspired by
[Resque](http://github.com/defunkt/resque), [Flickr's Offline Task
system(s)](http://code.flickr.com/blog/2008/09/26/flickr-engineers-do-it-offline/),
and others.

Harrison is named for [John
Harrison](http://en.wikipedia.org/wiki/John_Harrison), creator of the
[Longitude Clocks](http://en.wikipedia.org/wiki/Marine_chronometer), his
attempt to win the [Longitude
Prize](http://en.wikipedia.org/wiki/Longitude_prize).

## Design Goals

* tasks are deliberately time-limited (duration TBD) to encourage breaking them
  up into bite-sized chunks.
* queued tasks will always be run
* tasks must be idempotent, as they may be run more than once
* incomplete (failing) tasks will be retried until a retry limit has been
  reached
* tasks may be scheduled in the future
* tasks with matching arguments are considered duplicate tasks
* HTTP and JSON are used for maximal interoperability

## Task definitions

The "public" (initiator- and worker-facing) JSON description of a task looks
like:

    {
        "class": "UserBackfill",
        "args": [
            "12"
        ]
    }

This is the payload that is both used to create tasks and passed to worker
processes. _an `id` parameter may be included when sent to a worker._

The internal task definition looks like this:

    {
        "class": "UserBackfill",
        "args": [
            "12"
        ],
        "id": 42,
        "attempts": 0,
        "state": "ready",
        "lastError": null,
        "queuedAt": "2010-07-08T18:02:23.347Z",
        "firstRunAt": "2010-07-08T18:04:00.000Z",
        "reservedAt": "2010-07-08T18:03:00.000Z",
        "lastRunAt": null,
        "firstScheduledFor": "2010-07-08T18:02:23.347Z",
        "scheduledFor": "2010-07-08T18:02:23.347Z",
        "lastRunBy": null,
        "priority": ""
    }

### States

The following states are valid:

* ready
* reserved
* running
* error
* failed
   * TODO break into multiple reasons
   * timed-out

## Task ids and uniqueness

Task ids are generated by `INCR`ing the string value `next.task.id`. Uniqueness
is checked by taking the SHA of the public task description (sans whitespace)
and seeing whether a matching task has already been scheduled (and not yet run).

## Redis Keys

The following Redis data structures are used (prefixed with a user-defined
value, `harrison` by default):

* `next.task.id` - string value containing the maximum task id, suitable for
  `INCR`ing.
* `tasks:<id>` - hash containing the internal definition of a task
* `tasks` - set containing ids of all pending / running tasks (for uniqueness
  checks)
* `queue` - sorted set containing ids of pending / running tasks (sort key =
  `scheduledFor`)
* `by_priority` - sorted set containing ids of pending / running tasks with a
  particular priority (sort key = `priority`)
* `reservoir` - sorted set containing ids of tasks scheduled for the future
  (`scheduledFor` > now) (sort key = `scheduledFor`)
* `failed` - sorted set containing ids of failed tasks (sort key = `attempts`)
* `pending` - sorted set containing ids of reserved (in Harrison's local buffer)
  tasks (sort key = `reservedAt`)
* `error` - set containing tasks that passed the retry limit
* `invalid` - set containing ids of invalid tasks (invalid JSON, etc.)
* `errors:<id>` - string value containing the most recent error output for a
  failed task

## Metrics

* lifetime (completion time - `queuedAt`)
* time-to-completion (completion time - `firstRunAt`)
* time-to-failure (retry expiration - `queuedAt`)
* runtime (completion time - `reservedAt`)
* latency (time started - `firstScheduledFor`)
* number of attempts required
* number of tasks by queue
* number of tasks by queue by state
* number of tasks by priority
* number of tasks by priority by state
* number of tasks by queue by priority by state
* number of tasks by state
* number of waiting tasks
* tasks queued/second
* tasks run/second

## Web Interface

TDB

## Mapping Tasks to Workers

    harrison.WORKER_MAP = {
        "backfill": {
            "href": "http://localhost:8080/jobs/backfill",
            "concurrency": 10
        },
        "load": {
            "href": "http://localhost:8081/",
            "concurrency": 12
        },
        "echo": "http://localhost:8081/"
    };

## Intervals / Periodic Actions

The following actions need to occur on a regular basis:

* buffering waiting tasks into `pending` and Harrison's local queue
* running locally buffered tasks
* draining ready tasks from the reservoir to the main queue
* moving failed tasks (retry limit exceeded) to `error`
* collecting metrics

## Events

When a task completes (the HTTP request ends), some of the following actions
should be taken:

* add/update `failed` and reschedule into `reservoir` as
  appropriate
* remove from `pending`
* remove from `tasks` and `tasks:<id>` as appropriate
* update metrics

## Concurrency Controls

The concurrency of the following operations is configurable:

* number of running tasks per queue

## Dependencies

Harrison has the following dependencies:

* Node.js >= 1.100
* Redis 2.0.0RC1+
* [chain-gang](http://github.com/technoweenie/node-chain-gang)
* [ejs](http://github.com/visionmedia/ejs)
* [express](http://github.com/visionmedia/express)
* [expresso](http://github.com/visionmedia/expresso)
* [hashlib](http://github.com/brainfucker/hashlib)
* [redis-node-client](http://github.com/mojodna/redis-node-client)

## Installing

Follow the instructions for hashlib to add it to Node's `require.path`.

## Running the tests

    $ NODE_PATH=lib expresso

## Starting

    $ NODE_PATH=../../fictorial/redis-node-client/lib/:lib/ node lib/harrison.js
