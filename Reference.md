# The Hubiquitus Reference Guide

> IMPORTANT NOTICE: this document in under active writing

This document describes the internals of the Hubiquitus framework.

## Technical design

### Everthing is an actor

The Hubiquitus design follows the 'everything is an actor' philosophy, meaning that every Hubiquitus apps are made of actors, thus complying with the [Actor Model](http://en.wikipedia.org/wiki/Actor_model)) paradigm.

#### The actor model

**An actor is a form of lightweight computational entity that sequentially process incoming messages received on its inbox**

Actors in Hubiquitus comply with the fundamental principles of an actor: 

* each actor has an **inbox**, a kind of FIFO queue into which other actors can post messages to be processed,
* each actor has its proper **behavior** that is triggered sequentially for each message received in its inbox,
* each actor maintains its own **state** that it doesn't share with anyone else ("share nothing" principle); this state can be modified as the actor processes incoming messages.
* each actor can itself send **messages** to other actors; posting message is asynchronous so that it never blocks the process in which the actor is running,
* each actor can create **children** to which it will then be able to post messages as to any other actor.

The following figure summarizes these principles:

![actor model](https://github.com/hubiquitus/hubiquitus-reference/raw/master/images/ActorModel.png)

#### NodeJS as a container for actors

Hubiquitus is basically an implementation of the actor model for the great [NodeJS](http://nodejs.org) evented programming platform.

NodeJS is a natural choice as a container for actors since it provides features that comply with many aspects of the actor model:

* **Asynchronous evented I/O**: NodeJS allows binding *functions* to specific I/O events - such as "bytes has been written to this socket" - without blocking the execution thread until it occurs. This provides a simple and elegant way to implement the mechanism of the actor's inbox.
* **Single threaded execution**: each NodeJS process run programs using a single execution thread, we are sure to never have to deal with concurrency issues.
* **Child processes**: NodeJS natively supports creating forked process that communicates with their parent process using sockets, so that creating child actors as child processes becomes trivial.
* **First-class functions**: since it relies on the JavaScript programming language, NodeJS allows passing functions as parameters so that passing a behavior as a function to an actor is also trivial.

NodeJS provides some interesting building blocks to implement actors, but lots of things remain to be designed.

#### Actor's name and address

Since actors communicate using messages, we need to know their *name*  and their *address* so we can can properly deliver these messages to their expected recipients.

**Each *name* or *address* MUST be unique inside a group of actors that need to collaborate.**

Hubiquitus adopts the following IETF standards for formating the *name* and *address* of an actor:

* each *name* SHOULD comply with the [**Uniform Resource Name**](http://tools.ietf.org/html/rfc2141) IETF standard. For example, the following string is a valid actor name: `urn:hubiquitus.org:johndoe`.
* each *address* MUST comply with the [**Uniform Resource Location**](http://tools.ietf.org/html/rfc3986) IETF standard. For example, the following string is a valid actor's *inbox* address: `http://hubiquitus.org/johndoe`

> Please notice that precedent versions of Hubiquitus (until v0.5 included) used the JabberID semantics for the names of the actors. At the time of writing, Hubiquitus do not controls the precise format of the name.

#### Introducing *trackers*

While sending a message to an actor, the sender do not necessary needs to know the *address* of its *inbox*. The only thing it needs to know is its *name*. 

The Hubiquitus framework provides a special kind of actor that is called a ***tracker*** that acts as a **directory of actors**:

* When an actor is starting, it **registers itself to a *tracker***, providing its name and addresses.
* When an actor sends a message, it needs to **query the *tracker* for an address that match the name of the recipient**

Like any other actor, a tracker can register to another tracker so that an address can be resolved accross multiple trackers. This mechanism allows federating multiple groups of actors together. 

### Topology of Hubiquitus apps

#### From actors to apps - the 'russian dolls'

The structure of Hubiquitus apps take the form of a "russian doll" with four nested levels:

* `actor`: actors are the smallest building part of an application; they implement the elementary blocks of logic that are necessary to implement the features of the app,
* `process`: actors live in single-threaded processes; a single process can host an unlimited number of actors, as far as there's a sufficient amount of memory available.
* `program`: for various reasons, actors may be distributed on multiple processes running on a same host. for example, the child of an actor can be hosted in a forked process. These linked processes constitute what we call a program.
* `application`: hubiquitus apps are distributed applications that involve potentially many programs and many hosts

The following figure summarize this topology:

![hubiquitus exec model](https://github.com/hubiquitus/hubiquitus-reference/raw/master/images/HubiquitusExecModel.png)

#### The root and the forest

We said that a process hosts multiple actors, but we need to be more precise: a process host only one `root actor` which itself potentially creates somes children, which themselves potentially create grandchildren, which themselves…and so one.

We can say that **each process hosts the root of a tree of actors**. With the multiple processes it involves, **an application can be see as a forest of actors**.  

#### Child actors and child processes

Actors are free to create as many child actors they want. These actors can either be created:

* **in the same running process**: the child actor is created in the same process as its parent
* **in a new process**: the child actor runs in its own process that has been forked from the  parent process.

In both cases, child actors are stopped and destroyed when the parent actor's process stops.

> note: to come in future versions (i) the ability to host a child in a process that already exists (ii) the ability to host a child actor in a processs that is not a "child" process

## Implementation details


### The `hactor`object

Most of the Hubiquitus magic lay behind a single JavaScript objet called `hactor` that defines the structure common to every Hubiquitus actors:

* `actor id`: a **unique** key that identifies the actor, a simple string formatted like an URN,
* `behavior`: a **Javascript calllback** that is fired each time the actor receives a message,
* `endpoints`: a set of **addresses** onto which the actor will accept incoming messages
* `state`: an in-memory object that holds the state of the actor and onto which the behavior can make reads and writes,
* `children`: a list of references to child actors that could have been created by this actor
* `trackers`: a list of references to 'tracker' actors, a special actors that maintain the address book of all the actors

### Your first actor

All you have to do to create an actor is to instanciate this object with the following parameters:

* a unique actor **ID**
* a list of **one or more URLs** - the actor's inbound endpoints - that the actor will listen to for incoming messages,
* a **custom JavaScript callback** - the actor's behavior - that will be fired each time a message is received.

Once instanciated, you just have to start your actor and begin sending it messages, *et voilà* !

```js
// 	Instanciate your actor
var MyActor = require('hubiquitus').hactor(
	{ id: "myactor@localhost", in:["tcp://*.8888"]},
	function(err, message){
		// code your behavior below
	});

// Starting your actor
MyActor.start();
```

#### Inbound adapters

When you start your actor, the "hactor" class will first select, instanciate and start the inbound adapters that match the endpoint URLs you have specified.

Inbound adapters are special objects that are responsible for:

* listening to the port you have specified using the transport protocol you have specified
* registering the actor's behavior (remember, the custom callback you specified) to further I/O events (incoming messages)