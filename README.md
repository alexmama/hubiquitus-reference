# The Hubiquitus Reference Guide

## Table of contents
TODO

## The Hubiquitus actor engine

### About the actor model

An actor is a form of lightweight computational entity that sequentially process incoming messages received on its inbox.

Actors in Hubiquitus comply with the fundamental properties of an actor as defined by the Actor Model:

* each actor has an inbox, a kind of FIFO queue into which other actors can post messages to be processed,
* each actor has its proper behavior that is triggered sequentially for each message received in its inbox,
* each actor maintains its own state that it doesn't share with anyone else ("share nothing" principle); this state can be modified as the actor processes incoming messages.
* each actor can itself post messages to other actors; posting message is asynchronous so that it never blocks the process in which the actor is running,
* each actor can create children to which it will then be able to post messages as to any other actor,

### Why we choosed NodeJS

Hubiquitus provides a pure implementation of the actor model for the NodeJS evented programming platform.

NodeJS is a great choice as a core to implement the actor model since it provides features that comply with many aspects of the actor model:

* Single threaded execution: since NodeJS it running programs in a single thread, we are sure to never have to deal with concurrency issues.
Asynchronous evented I/O: NodeJS allows a program to register to specific I/O events - such as "bytes has been written to this socket" - without blocking the execution thread until it occurs. This provides a simple and elegant way to implement actors inboxes.
Child processes: NodeJS natively supports creating forked process that communicates with their parent process using an IPC socket channel, so that creating child actors becomes trivial.
First-class functions: since it relies on the JavaScript programming language, NodeJS allows passing functions as parameters so that passing a behavior to a child actor is also trivial.
Hubiquitus actors - the short story

### The "hactor" class

Most of the Hubiquitus magic lay behind a single JavaScript objet called "hactor" that defines the structure and lifecycle common to every Hubiquitus actors.

All you have to do to create an actor is to instanciate this object with the following parameters:

* a unique actor ID
* a list of one or more URLs - the actor's inbound endpoints - that the actor will listen to for incoming messages,
* a custom JavaScript callback - the actor's behavior - that will be fired each time a message is received.

Once instanciated, you just have to start your actor and begin sending it messages, et voilà !

	// Instanciate your actor
	var MyActor = require('hubiquitus').hactor(
		{ id: "myactor@localhost", in:["tcp://*.8888"]},
		function(err, message){
			// code your behavior below
		});

	// Starting your actor
	MyActor.start();

### Inbound adapters

When you start your actor, the "hactor" class will first select, instanciate and start the inbound adapters that match the endpoint URLs you have specified.

Inbound adapters are special objects that are responsible for:

* listening to the port you have specified using the transport protocol you have specified
* registering the actor's behavior (remember, the custom callback you specified) to further I/O events (incoming messages)

The figure below summarizes these principles:

Authors and Contributors

@alexmama