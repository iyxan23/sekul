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

# Introduction

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

## The `req` Message Type

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

### Arguments in a `req`

A `req` can have arguments in the form of a key-value store similar to how JSON functions.
Its types are provided by the server in a [`ContextChange event`](#context-change-event).

Here's an example of requesting a product with a specific id.

```
-> req  Product { id: 10 }
<- resp Product { id: 10 } = { title: "Plastic Bag", desc: "Just a plastic bag, nothing special", price: 1 }
```

### Idempotence & Caching of a `req`

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

## The `resp` Message Type

The `resp` message type is a message sent by the server as a response to a
[`req` message type](#the-req-message-type) from the client.

In this doc, I will write `resp` message types like this:

```
-> req  Product { id: 10 }
<- resp Product { id: 10 } = { title: "...", desc: "...", price: 5 }
```

`resp` are written with its corresponding `req` notation right before the equal sign.
Notice how req has the content `Product { id: 10 }` and resp has the same data, but
it has "resolved" the resource which is indicated by the equal sign, followed by its
resource's actual data.

### Caching of a `resp`

As we've discussed previously in [`req`'s Idempotence and Caching](#idempotence-caching-of-a-req),
the indication that we've talked about not only mean that the server "resolved" the
request that the client sent. But also a "cache key" for the client to save for later
use.

### Payload of a `resp`

The payload of a `resp` message is similar to a `req`'s key-value store style, JSON-like
structure. Though a `resp` could include some special values: `req`, `resp` and `disp`.

Yes, it could include message types. For what purpose? Well, let's see them in the following
request-response flow:

```
-> req  ProductList {}
<- resp ProductList {} = [req Product { id: 10 }, resp Product { id: 11 } = ...]
```

Woah, what's this all about?

With SEKUL, you could define relations using types. Here, `resp ProductList` returns a
`Product[]`, and if you've known, `Product` is a possible `req` (or a defined resource
that you could query to the server).

Here, the server resolves `req ProductList` to be
`[req Product { id: 10 }, resp Product { id: 11 } = ...]`, which means that
`resp ProductList` returned values that could be retrieved using the same `req` (see
the `req Product`). We could send in a `req` message with the value from ProductList
to get the Product as we wish.

But the server could also opt in to send the "fetched" product as well, so the client
won't need to do a `req` anymore (see the `resp Product`). Notice how the `resp`
included the parameters needed to fetch the same data (`{ id: 11 }`). So, we could
cache this `resp` as if we've did a `req` to it before! But doing it without any
further requests to the server!

A bit confused? Here's an example

```
-> req  ProductList {}
<- resp ProductList {} = [req Product { id: 10 }, resp Product { id: 11 } = ...]

 | the client cached the following:
 |  - ProductList {}     = [Product { id: 10 }, Product { id: 11 }]
 |  - Product { id: 11 } = ...

 | notice how the client also cached `Product { id: 11 }` with the value provided by
 | the 2nd value from the array of `ProductList`

 | suppose the client wanted to see the 1st Product

-> req  Product { id: 10 }
<- resp Product { id: 10 } = ...

 | the client cached the following:
 |  - ProductList {}     = [Product { id: 10 }, Product { id: 11 }]
 |  - Product { id: 11 } = ...
 |  - Product { id: 10 } = ...

 | nice isn't it?
```

Cool! You've just understood one of the cool principles of SEKUL.

In summary, including a `req` object inside a `resp`'s content indicates an unfetched
data but related to that "object". Including a `resp` inside a `resp`'s content
basically prefetches it for the client, so the client wouldn't need to perform a `req`
to know its value.

#### Referencing an Argument

Sometimes a `req` that we get from a `resp` might be a "derivative" of the `resp`'s original
`req`, where it might use its original original. For example, a `CartItem` resource that is
a child of a `Cart` resource. Here's an example:

```
-> req  Cart { id: "..." }
<- resp Cart { id: "..." } = {
            title: "...",
            items: [
              req CartItem {
                parent: args.id,
                idx: 0
              },
              req CartItem {
                parent: args.id,
                idx: 1
              }
            ]
          }

 | the client now knows that the `req CartItem` from resp `Cart` is correlated with `Cart`'s
 | argument named "id".
```

This feature is useful for `disp`, which we will later discuss.

> I'm also thinking of an idea where we could reference the response's fields, not just
> the arguments given on a `req`. So something like
>
> ```
> resp Cart { id: "..." } = {
>          someId: "hey",
>          items: [
>            req CartItem {
>              parent: .someId, // will reference the "hey"
>              idx: 0
>            }
>          ]
> ```
>
> But dunno if it actually is useful or not

#### `disp` in `resp`

Assuming you've read the next section ahead about [`disp`](#the-disp-message-type), a `resp`'s
response could also include a disp of which the client could "fire" to the server. You could say
they work quite the same when compared to methods in OOP.

The thing of how it works is similar to how a `resp` includes a `req`/`resp` in its contents.

Here's an example:

```
-> req  CartItem { parent: "...", idx: 0 }
<- resp CartItem { parent: "...", idx: 0 } = {
             delete: disp DeleteCartItem { cartParent: args.parent, idx: args.idx },
           }
```

The client could know what kind of dispatch to do what kind of action. It doesn't have to know
the `disp`'s opcode or name, it could know that from a `resp`'s content (here, accessing the
`delete` from the `resp` leads us to the disp that we needed to actually delete the `CartItem`).

The client could just:

```
-> disp DeleteCartItem { ... }
```

And done, the cart is deleted.

> Maybe also make deletion possible in caches? I haven't thought of that yet.
> How can the client know that a CartItem got deleted? And how can it update its cache?

#### Empty Parameters in a `req`/`disp` of a `resp`

We could define a parameter that shall be filled manually by the client when doing a `disp` or
a `req`. This empty parameter is especially useful for `disp` messages. Here's an example:

```
-> req  Product { id: "..." }
<- resp Product { id: "..." } = {
            name: "Laptop",
            desc: "A shiny laptop",
            category: req Category { id: "..." },
            changeCategory: disp ChangeCategory {
              productId: args.id,
              target(string)    // <- target is an empty parameter, the client must fill it by itself
            },
            rename: disp RenameProduct {
              productId: args.id,
              newName(string)    // <- target is an empty parameter, the client must fill it by itself
            },
         }
```

## The `disp` Message Type

Unlike the previous `req` and `resp` message types, `disp` or dispatch is an **asynchronous**
message type.

Dispatch is similar to a regular req, but the difference is that the server does not give
any exact response. It is asynchronous. Dispatches are usually used for actions that are
performed by the user (the actual user) using the client. So things like clicking a button
or moving their cursor around. Those actions should fire a dispatch to the server.

Think of it more like a reverse-event, and the client is "emitting" an event to the server.
It has a different name so its easier to differentiate one another.

Here's how a `disp` looks like:

```
-> disp MoveFile { from: "/abc", to: "/def" }
```

It's quite straightforward. The only difference is that a disp usually does not have a
response. Though, the server shall send [`event` messages](#the-ev-message-type) instead.

## The `ev` Message Type

An event message is an asynchronous message that comes from the server. It has a name and
could include payloads. The server could send as much as it'd like, and the client could
ignore or dismiss events as they like.

```
<- ev   Notification { title: "You've got mail!", description: "Something something" }
<- ev   Notification { title: "New offer", description: "You got 99% off of something" }
```

### Special `ev`: Update Events (`evup`)

One of the cool things about SEKUL is that both resources fetched using a `req` could
react to changes in response to a special update event.

```
-> req  ProductList {}
<- resp ProductList {} = [req Product { id: 1 }, req Product { id: 2 }]

 | client cached the following:
 |  - ProductList {} = [req Product { id: 1 }, req Product { id: 2 }]

 | sometime later

<- evup ProductList {} = [req Product { id: 1 }, req Product { id: 2 }, req Product { id: 3 }]

 | client cached the following:
 |  - ProductList {} = [req Product { id: 1 }, req Product { id: 2 }, req Product { id: 3 }]
 |
 | and the client would update the listing to reflect the update event.
```

An `evup` message's primary goal is to notify the client of changes in a resource.
It could be anything, ranging from something like `req CurrentNote` of a note-taking app,
where the user could dispatch something like `disp SelectNote` to change notes.

# Further Features

- [System API](./system-api.md)
  - [Context API](./context-api.md)
  - [Subscription API](./subscription-api.md)
- [Layers of Architecture](./layers-of-architecture.md)
- [Use Cases](./use-cases.md)

# Background

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
