# SEKUL

Welcome to the beginning of a shiny new simple protocol to manage realtime connections.

In short: It's a protocol that supports caching by default, type-safe realtime, and
really extensible. Targeted for realtime apps similar to how Figma works, but can
also be used in any case imaginable.

You could say it's functionality to be similar when compared to regular HTTP REST APIs,
but it's constraints are clearer and connections between objects (e.g. `/product/list`,
`/product/10`) makes sense. Also don't forget the type-safetiness it comes with!

When compared to SOAP, it's actually not that far off. The only difference is that it's
way simpler, and way easier. Plus the added ability to transfer and exchange realtime data.

Currently there's only documentation form of work of how this protocol will actually
look like and work. I am planning to create rust crates to run SEKUL over HTTP and
WebSocket and also client libraries with wasm support. But that's for later~

## Introduction

SEKUL uses a concept of **resources**, or **objects** in object-oriented fashion.

Anything is a resource, `Product`, `CurrentFrame`, `Properties`.

Types of messages are categorized into two categories: Synchronous, and Asynchronous.
There are 4 types of messages in SEKUL:

- `req` message type (request) (synchronous)
- `resp` message type (response) (synchronous)
- `disp` message type (dispatch) (asynchronous)
- `ev` message type (event) (asynchronous)

We will discuss them one-by-one, referencing to other types of messages as it is
a bit circularly-dependent.

### The `req` Message Type

The `req` message is basically a request. It's like the client asking for something
to the server. In this doc, I will write `req` messages to be like this:

```
req Product {}
```

This is a request for an object named `Product` with 0 arguments (hence the empty `{}`).
The server will in turn send back a [`resp` message type](#the-resp-message-type) like
so: `resp Product = ...`.

A request-response flow would look like this:

```
-> req  Product {}
<- resp Product {} = { title: "Gaming Mouse", desc: "Very cool gaming mouse", price: 25 }
```

The `->` indicates a message being sent from the client to the server. And the `<-`
to be the opposite.

Read more about the [`resp` message type here](#the-resp-message-type).

#### Arguments

A `req` can have arguments in the form of a key-value store similar to how JSON functions.
Its types are provided by the server in a [`ContextChange event`](#context-change-event).

Here's an example of requesting a product with a specific id.

```
-> req  Product { id: 10 }
<- resp Product { id: 10 } = { title: "Plastic Bag", desc: "Just a plastic bag, nothing special", price: 1 }
```

#### Idempotence & Caching

The awesome thing about SEKUL is that resources and/or objects are cached by default.
Each time a `req` is made, the client must store its corresponding `resp` object in a
its memory, complete with "instructions on how to perform the same req" (basically
its name and its arguments).

**`req`s are idempotent. It can not return anything else if the same argument is passed.**

And that makes the perfect environment for caching to breed. The client wouldn't need
to send the same `req` to get the response they needed if they have done it before.

Though, if the client needed to, it could send the same `req` even though it had done
it before (it must've cached it before), and the server should in turn response with
the appropriate resource.

### The `resp` Message Type

The `resp` message type is a message sent by the server as a response to a
[`req` message type](#the-req-message-type) from the client.

I will write `resp` message types like this:

```
<- resp Product { id: 10 } = { title: "...", desc: "...", price: 5 }
```

## Background

Well, lately my abandoned project named [`dalang`](https://github.com/iyxan23/dalang)
took most of my braincells during my free time in my boarding school. I still had the
urge of creating a really cool video editor complete with multiplayer, realtime editing
support, and server rendering. It was a really cool project and I could never leave it.

I couldn't have much liberal access to my laptop to code, so I decided to maybe try to
create some foundation for now with good ol' pen and paper. I believe that with thinking
some of the architectures thoroughly, I could use them later on when I could use my laptop
more freely. I started out in the websocket side.

Current dalang's way of communicating didn't.. work.

It relies on MessagePack instead of something like JSON, which was quite cool. Both
dalang's client and server communicates in the same way, using packets with opcodes.

So the idea is, let's say we have an opcode of `Login` which is sent by the client.
It has the types of `{ username: string, password: string }`
