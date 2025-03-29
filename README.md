## SEKUL

> Stateful rEaltime & reaKtive commUnication protocoL

A transport-agnostic stateful protocol with a focus on caching and efficiency.

This project is mostly for fun and learning, I don't exactly know what kinds of
benefits it may bring in an actual practical scenario. But that's an idea worth
researching for later.

It's intended use is for stateful apps that need to send/receive data in
real-time and that could benefit a lot from simple caching, like my upcoming
[dalang](https://github.com/iyxan23/dalang) video editor web-app; which just so
happens to be the

### Quick Overview

What problem is SEKUL trying to address?

The idea of SEKUL was first conceived to address the inconsistencies often
riddled in apps using HTTP as a REST API. The uncertainty and loosely typed
nature of HTTP APIs can make it difficult to build robust apps.

Technologies like OpenAPI already existed in the industry to tackle this issue.
However, I wanted to explore a different approach; what if I designed my own
protocol? What would it look like?

From here on out, the idea evolved to tackle more complex and interesting
problems, some of which may not be practical for general use. Again, this
project really is just for fun and exploration; a stepping stone, if you will,
toward my bigger project [dalang](https://github.com/iyxan23/dalang).

### Project Structure

This project contains a few crates:

- `skl-core`

  Common code used by the other crates.

- `skl-server`

  Server-side code that contains logic to handle the "business logic" of the
  protocol. It does not specifically implement any single transport protocol,
  as the protocol itself is intended to be transport-agnostic.

- `skl-client`

  Same as `skl-server`, but for the client-side.

- `skl-ws`

  The WebSocket transport adapter implementation of sekul.

### Inspirations

SEKUL could never come to fruition without my exposure to these ingeninous
projects:

- [**React**](https://react.dev): really made me think of idempotency and how huge "function purity" really is
- [**trpc**](https://github.com/trpc/trpc): for the transport-agnostic idea
- [**Freedesktops' D-Bus**](https://dbus.freedesktop.org/): yup, it's weird, but it's intriguing how they have `org.freedesktop.DBus.Introspectable` to list method/interface/signal names; this is going to SEKUL

### Learn More

Interested to know more? Please head on to [docs/](docs/). Apologies for the
terrible writing, the docs were written a few years prior. I have plans of
rewriting to make them clearer and exact in the future.
