# Funcatron

Serverless on Your Cluster:

* Define Endpoints in Swagger
* Write Functions in Java, Scala, Clojure, Python, or JavaScript
* Deploy to Mesos, Kubernets, or Swarm
* Autoscale

This document sets out the goals for the [Funcatron](http://funcatron.org) project.

## What's Funcatron

Amazon's [Lambda](https://aws.amazon.com/lambda/) popularized
["serverless"](http://www.martinfowler.com/articles/serverless.html)
code deployment. It's dead simple: associate a "function" with an event.
Each time the event happens, the function is applied and the function's
return value is returned to the event source. An event could be an HTTP(S)
request, something on an event queue, whatever.

Functions are ephemeral. They existing for the duration of the application.
Once the function returns a value, all of it's state and scope and everything
else about it is assumed to go away.

Scaling this kind of architecture is simple: the more frequently a function gets
applied, the more compute resources are allocated to support the function...
and [Bob's Your Uncle](https://en.wikipedia.org/wiki/Bob%27s_your_uncle).

The current popular function runners (competitors to Amazon's Lambda) are
proprietary: when you write to the API for Lambda or Google's
[Cloud Functions](https://cloud.google.com/functions/docs/),
you're locked into that vendor.

There's currently no (well, there's [OpenWhisk](https://developer.ibm.com/openwhisk/))
generic way to do the auto-scale function thing on a private cloud or in a
way that can migrate from one cloud provider to another.

Funcatron addresses this. Funcatron is a cloud-provider-neutral mechanism for
developing, testing, and deploying auto-scalable functions.

Funcatron is designed to run on container orchestration clusters:
[Mesos](https://mesosphere.com/), [Kubernetes](http://kubernetes.io/), or
[Docker Swarm](https://docker.com).

## Software Lifecycle

Software goes through a lifecycle:

- Authoring
- Testing
- Staging
- Production
- Debugging

A key driver for Funcatron is to address software at each stage of the lifecycle.

### Authoring

An engineers sits down to write software. The faster the turn-around between
code written and "trying it out," the more productive the engineer will be.

Funcatron supports a "save, reload" model where the engineer saves a file
(presuming they're using an IDE that does compilation on save or are using a
non-compiled language), and their function endpoints are available. That's it.
No uploading. No reconfiguration. No waiting. The endpoint is available on save.

Further, that Funcatron model requires knowledge of two things:
[Swagger](http://swagger.io) and how to write a simple function in Java, Scala,
Clojure, Python, or JavaScript. That's it.

The engineer defines the endpoints in Swagger and uses the `operationId` to
specify the function (or class for Java and Scala) to apply when then endpoint
is requested. Funcatron takes care of the rest.

Between the fast turn-around and low "new stuff to learn" quotient,
it's easy to get started with Funcatron. It's also easy to stay productive
with Funcatron.

Also, developers need only have Docker installed on their development machine
to live-test Funcatron code.

### Testing

Because Funcatron endpoints are single functions (or methods on newly
instantiated classes), writing unit tests is simple. Write a unit test and
test the function.

### Staging

Funcatron code bundles (Funcs) contain a Swagger endpoint definition and the functions
that are associated with the endpoint and any library code. For JVM languages,
these are bundled into an Uber JAR. For Python, a [PEX](https://github.com/pantsbuild/pex)
file. The Swagger definitions for an endpoint are unique based on
host name and root path.

Funcatron supports performing [`sed`](https://en.wikipedia.org/wiki/Sed) operations
on Swagger fields based on environment variables. This allows rewriting host name,
root path, and other Swagger fields if the Func is deployed to a staging (or test)
server. Thus, there's one deployment unit (a Func bundle) that have well defined
behaviors across test, staging, and production servers.

### Production

Funcatron allows simple deployment and undeployment of end-point collections
defined in Swagger files and implemented in a JVM language, Python, or NodeJS.

Requests are forwarded from Nginx via a message queue to a dispatcher (a Tron).
Based on the hostname and root path, the message is placed on a queue for
a specific Func. The Func processes the request and sends the response
to a reply queue. The Nginx process dequeues the response and returns
it as an HTTP response.

The number of Func instances running on a cluster is based on the queue depth
and response time. The Func manager sends statistics back to the Trons
and the Trons change Func allocation based on these statistics by
communicating with the container orchestration substrate (Mesos, Kubernetes,
Swarm) and changing the allocation of Func running containers.

From the DevOps point of view: deploy a Func and it binds to the appropriate
HTTP endpoint and scales to handle load.

### Debugging & Test Cases

Funcatron logs a unique request ID and the SHA of the Func with every log line
related to a request. This allows correlation of requests as they fan out through
a cluster.

Funcatron allows dynamic changing log levels on a Func-by-Func basis which allows
capturing more information on demand.

All communications between the front end, Funcs, and back again are via well
defined JSON payloads. Funcatron allows capturing request and response
payloads on a Func-by-Func basis (complete streams, or random sampling).
This data can be used for testing or debugging.

## Architecture

Funcatron has some ambitious goals... and has an architecture to facilitate
achieving these goals.

In all but development mode, Funcatron runs on a Docker container orchestration
system: Mesos, Kubernetes, or Docker Swarm. We call this the "container
substrate." Each of the Funcatron components can be scaled independently with
messages to the container substrate.

For HTTP requests, Funcatron uses Nginx and Lua (via the
[OpenResty](http://openresty.org/en/) project) to handle the requests. A small
Lua script encodes the request as a payload that's sent to RabbitMQ. For large
request or response bodies, the body will be written to a shared distributed
filesystem (e.g., HDFS) and a reference to the file will be enqueued to
RabbitMQ. For all but the highest volume installations, 2 Nginx instances
should be sufficient.

RabbitMQ is deployed on the cluster. RabbitMQ has well understood scaling
characteristics and can handle high traffic loads.

A "Tron" module dequeues the request from Nginx. Based on the queue depth
for the "request firehose", the container substrate can launch more Trons.

Based on the combination of `host` and `pathPrefix` attributes in the Swagger
module definition, the Tron enqueues the request on the appropriate queue.

A runner module dequeues messages from a number of host/pathPrefix queues and
forwards the request to the appropriate Func. The runner then takes the function
return value and appropriately encodes it and places it on the reply queue which
dequeued by the original Nginx instance.

Each Func can run multiple modules. Based on queue depth, queue service time,
and CPU usage stats from the Funcs, more runners can be allocated on the substrate,
or more Funcs can be allocated across the runners.

The Lua scripts dequeues the response and turns in into an Nginx response.

Because all of the operation of the Funcs and Trons can be captured as messages
(and all the messages are in JSON), it's possible to capture message streams for
testing and debugging purposes.

Every request has a unique ID and each log line includes the unique ID so it's
possible to correlate a request as it moves across the cluster.

## Project Status

The project is in the initial development phase. Java and Scala Func bundles
are currently supported.

## Licenses and Support

Funcatron is licensed under an Apache 2 license.

Support is available from the project's founder,
[David Pollak](mailto:feeder.of.the.bears@gmail.com).
