# GearMonkey

GearMonkey is, at its most basic, a simple API schema built on top of a data store
whose built-in functions allow it to function as an asynchronous job server.
 
Based heavily on [Gearman](http://gearman.org/), GearMonkey allows processes acting
as _Clients_ to request jobs from dedicated processes referred to as _Workers_.
This aims to create a language-independent, easily networked Microservice architecture
where an individual function can be farmed out to the environment best suited to complete it.

Although these specifications are written with [Redis](http://redis.io/) as the backend
server application, the underlying principles should work on any data store that is even
partially ACID-compliant:

* Store job data in an indexed data structure that can be quickly read from and updated.
* Maintain a [stack](https://en.wikipedia.org/wiki/Stack_(abstract_data_type)) of
  queued jobs that can be _atomically_ shifted (First In, First Out principle).
* Maintain a count of connected _Worker_ processes.
* Expose an interface where connected listeners are informed of state changes in
  the stored data.
  
## Disclaimer

* The interfaces described in this document serve to dictate _what features_ a GearMonkey
  implementation must provide, not _how_ it provides them. Method names, parameters, and
  return values can, and probably should, differ per individual implementation. Where one
  implementation might prefer to keep calls low-level, requiring only string parameters,
  another might offer a more Object-Oriented structure where a job is instanced as an
  object prior to being handled by the client instance. As long as a library correctly
  follows implementation instructions, it will be considered a GearMonkey actor.

## Specification

The GearMonkey architecture consists of three parts: _Servers_, _Clients_, and _Workers_.
These entities all work around the concept of _Jobs_.

* A _Server_ is a running Redis instance where requested jobs will be stored.
* A _Client_ is a process that submits a job request to the server, and optionally
  monitors its status to completion.
* A _Worker_ is a process that polls for queued jobs, attempts to complete them,
  and submits the result (or an error raised during job execution) back to the server.
* A _Job_ is any piece of work that needs to be completed, that the client chooses
  not to execute internally. A job has a name (referred to as the _Function_), and
  usually a payload consisting of one or more parameters, referred to as the _Input_.
  
**What should be considered a job?**

A Job can be _anything_. Even the most basic function calls can be processed as a job
through GearMonkey workers. However, efficient implementation implies that functions are
implemented as jobs only when:
 1. The client lacks the tools or resources to execute the function internally, or
 2. The function should be run asynchronously in order to optimize efficiency
    and end-user experience.

**Variable placeholders**

Throughout this specification, the following placeholders will be used to reference
values known only at runtime. Logically, implementations must substitute these placeholders
with their actual values.

* `{FN}` : A job's function name. (String)
* `{ID}` : A generated job ID. (Integer)
* `{PRIO}` : The job priority as set by the client. (String: [ low | normal | high ])
* `{OPTIONS}` : A set of 0 or more options that influence the client's behaviour.
  We currently define two options:
  * `OPT_BLOCKING` instructs the client to block further code execution until the job
    has been finished either successfully or with raised errors, effectively making the
    call synchronous.
  * `OPT_ROLLCALL` instructs the client to 'Roll Call' prior to job creation, and check
    if any workers are currently registered with the server.
* `{INPUT}` : The job's payload (String)
* `{OUTPUT}` : The finished job's return value (String)
* `{TIMEOUT}` : The timeout (in seconds) for blocking calls as set by the client. (Integer)

### Client

#### Interface

In order to serve as a GearMonkey Client, a library must expose two methods to
userspace:

**`Client.run( {FN} , {INPUT} , {PRIO} , {OPTIONS}, {TIMEOUT} ) ==> {ID}`**

Instruct the client to connect to the server, and deliver the job.
The client must be able to poll the job status afterwards, which means the method must
store or return the generated `{ID}` that the server responds with. If the server does
_not_ respond with a job ID, job creation must be assumed to have failed.
* The default priority is **normal**, and can be overridden in the method call.
* The default options set is empty, and can be overridden with any combination of the
  options mentioned above. The client implementation must honor these options, and
  change strategy accordingly.
  * On `OPT_BLOCKING`, the client must block further execution, and only return when
    the worker has finished execution and communicated the status back to the server.
    Note that this does not imply a successful execution: A job is also considered
    'finished' if execution is cancelled or interrupted due to raised errors.
  * On `OPT_ROLLCALL`, the client must perform a 'Roll Call' where it queries the number
    of workers currently registered for the specified function `{FN}`. If this query
    returns 0 or any non-integer value, the client must raise an error in userspace.
    A Roll Call does not imply that the job will be finished, or even started, quickly.
    Also, it does not imply atomicity, meaning that any number of workers can die off
    between the Roll Call and the actual job creation.
    
**`Client.get( {FN} , {ID} )`**

Instruct the client to return structured data that describes the current state of the job.
This data must include at least the output (`{OUTPUT}`) and the status (`{STATUS}`).
This data can be requested from the server in realtime, or can be preset by `Client.run`
in blocking mode.

#### Implementation

```
// Creating and inserting a new job:
// ---------------------------------

// (OPT_ROLLCALL only) Count registered workers. Raise an error when the result does not >= 1
GET count:{FN}

// Generate a job ID
INCR uid:{FN}

// Insert the job into a Redis hashmap. Set it to expire in 90000 seconds (25 hours).
HMSET job:{FN}:{ID} status "idle" input "{INPUT}"
EXPIRE job:{FN}:{ID} 90000

// Publish a created event to the channel.
PUBLISH channel:{FN} "create:{ID}"

// Insert the job ID into the queue.
LPUSH queue:{FN}:{PRIO} {ID}

// (OPT_BLOCKING only) Run a blocking Pop call on a custom list.
BRPOP lock:{FN}:{ID} {TIMEOUT}

// Redis must now return {ID} back to the client,
// if it hasn't yet done so with the INCR call.

// Fetching job data:
// ------------------

// Client.get will always return the full job hash
HGETALLL job:{FN}:{ID}
```

### Worker

#### Interface

In order to serve as a GearMonkey Worker, a library must expose two methods to userspace,
but is encouraged to include a third one:

**`Worker.accept( {FN} )`**

Instruct the worker to accept jobs within the specified function. A worker *may* accept
multiple functions, and so must accept multiple calls to `Worker.accept`, although using
this feature would considered a bad practice in _most_ cases.

**`Worker.work( {INPUT} )`**

A worker must require userspace to implement a 'working' function that the worker will
call when it recieves a job payload. This custom function must return a value on success,
or raise an error if it can not successfully complete. The worker will base the `{STATUS}`
on the result value of this function, and will set it to either "**success**" or "**error**".

**`Worker.update( {DIVIDEND} , {DIVISOR} )`**

This function is optional, but recommended for workers (and clients) often processing
jobs with a runtime of more than a few seconds. By calling this method from inside the
working code, a worker may notify clients whenever a job is partially completed.
It does so by sending the Dividend (indicating the currently completed part)
and the Divisor (the total workload) to the server. Non-blocking clients can poll
these values to calculate the overall progress.

#### Implementation

```
// On worker start:
// ----------------

INCR count:{FN}

// Main worker loop:
// -----------------

// Run this in an endless loop
BRPOP queue:{FN}:high queue:{FN}:normal queue:{FN}:low 1

// On recieving a job:
// -------------------

// Publish a 'start' event
PUBLISH channel:{FN} "start:{ID}"

// Update the job status
HMSET job:{FN}:{ID} status "busy"

// Sending intermediate status updates:
// ------------------------------------

HMSET job:{FN}:{ID} status:dividend "{DIVIDEND}" status:divisor "{DIVISOR}"

// On completing a job:
// --------------------

// Update the job hash
HMSET job:{FN}:{ID} status "{STATUS}" output "{OUTPUT}"

// Set the hash to expire in 25 hours
EXPIRE job:{FN}:{ID} 90000

// Publish a 'finish' event
PUBLISH channel:{FN} "finish:{ID}"

// Push to the lock list (to finish any blocking calls) with an expire of 10 seconds
LPUSH lock:{FN}:{ID} "OK"
EXPIRE lock:{FN}:{ID} 10

// On worker exit:
// ---------------

DECR count:{FN}
```