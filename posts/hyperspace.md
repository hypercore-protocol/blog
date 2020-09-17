---
title: "Hyperspace: A Minimal Hypercore Server"
description: Hypercore, batteries included.
date: 2020-09-17
first_author: Andrew Osheroff
first_author_page: https://twitter.com/andrewosh
tags:
  - hypercore
  - hyperspace
  - release
layout: layouts/post.njk
---
### Quick Primer

If you're already familiar with the Hypercore Protocol, feel free to skip over this bit. If not, here's a quick overview of our modules to get you up to speed.

The Hypercore Protocol consists of three things:
1. An append-only log structure (__Hypercore__) designed for fast, secure replication between peers in peer-to-peer networks. Hypercore is our core primitive.
2. A DHT implementation that works well in home networks (__Hyperswarm__). We mainly use this for discovering and replicating Hypercores.
3. A few higher-level data structures built on top of Hypercores (__Hyperdrive__, __Hypertrie__), for constructing filesystems and kv-stores.

Hypercore and Hyperswarm are the "foundational" pieces of the stack -- our other modules build on top of them. One such module, __Corestore__, provides a convenience layer for managing collections of Hypercores, which is useful in the higher-level data structures like Hyperdrive.

## Intro

Back in May, we released Hyperdrive v10, which introduced major new features, like indexing improvements and mounts. To get folks off the ground quickly, we paired v10 with a new Hyperdrive daemon, providing simple, out-of-the-box answers for managing drive storage and Hyperswarm networking. The daemon was designed for ease of use and diverse use-cases, bundling a CLI, a gRPC API, and FUSE support. We imagined `hyperdrive start` getting you 90% of the way to creating and sharing drives, without burdoning you with many technical details. If you're intested in how we approached this, check out our earlier [blog post](https://blog.hypercore-protocol.org/posts/announcing-hyperdrive-10/).

Our relationship with the daemon was bright but brief! It's been working (mostly) reliably in the [Beaker 1.0 Beta](https://beakerbrowser.com), and for the first time it let us ship a single, unified product. That said, after launch we began adding support for new Hypercore-based data structures, and we realized this would require both substantial refactoring of the daemon internals and extensions to the API. At the end of the day, the daemon was built with Hyperdrives in mind, and we saw a future of complexity and bloat in extending its scope in that direction.

We're always looking to simplify and streamline our stack, so with minimalism in mind we started experimenting with a more flexible implementation. Those experiments turned out nicely, and Hyperspace is the result.

## What is Hyperspace?

Hyperspace is a lightweight service that provides remote access to Hypercores/Hyperswarm, and nothing more -- where the daemon gave you `RemoteHyperdrive`, Hyperspace gives you `RemoteHypercore`. Since Hypercore and Hyperswarm are the foundational bits in our ecosystem, centering our primary service around them caters to a broader set of users. When compared to the Hyperdrive daemon, Hyperspace has a tiny scope, but keeping the scope small enables a vastly larger opportunities in userland.

As an example of where this simplicity shines, a "remote Hyperdrive" built on Hyperspace is actually just a regular old Hyperdrive, but with `RemoteHypercores` inside. Previously, adding support for a new Hyperdrive feature involved implementing the feature, updating API schemas, and adding methods to the daemon/client. With Hyperspace, new Hyperdrive features can live in Hyperdrive alone.

As with the daemon, Hyperspace handles storage and networking for you: the Hypercores it manages are all stored in a single location on-disk (inside the `~/.hyperspace/storage` directory), and the server keeps a long-running Hyperswarm node online for you. As far as features go, there really are only two:
1. `RemoteHypercore`, `RemoteNetworker`, and `RemoteCorestore` APIs served over a tiny RPC system. We'll never add anything beyond these, to keep things simple.
2. A Node.js client library

We've also gotten rid of PM2 in Hyperspace -- the service only runs in the foreground and isn't a daemon. This should make it easier to use with a wider variety of process managers, like systemd or a tray app.


### The API

We're constantly adding new features to the higher-level data structures (like Hyperdrive), but the Hypercore and Hyperswarm APIs are updated relatively infrequently. We can take advantage of that to keep the Hyperspace API as small and stable as possible. We also wanted the API to be clear and easy-to-use, so we opted to use a custom binary RPC system called [HRPC](https://github.com/mafintosh/hrpc) instead of gRPC.

Like gRPC, [HRPC service definitions](https://github.com/hypercore-protocol/hyperspace-rpc/blob/master/schema.proto) are written as Protocol Buffers schemas. Unlike gRPC, HRPC is transport-agnostic (you can easily pipe it over a WebSocket, for example) and has simpler built-in error handling, which assists in debugging. It's super easy to use, and it's a much smaller dependency to boot.

### The Client Library

We've shot for API parity between [`Corestore`](https://github.com/andrewosh/corestore), [`Hypercore`](https://github.com/hypercore-protocol/hypercore), the [`@corestore/networker`](https://github.com/andrewosh/corestore-networker) and their remote counterparts. As such you should be able to drop a `RemoteHypercore` into any module that currently consumes a `Hypercore`, and likewise for the other two.

As an example, updating Hyperdrive to use `RemoteHypercores` was just a matter of dropping in a `RemoteCorestore` as the first argument:
```javascript
const hyperdrive = require('hyperdrive')
const { Client: HyperspaceClient } = require('hyperspace')

// Assumes the Hyperspace service is already running.
const client = new HyperspaceClient()
const drive = hyperdrive(client.corestore())
```

### Trying it Out

First, install hyperspace globally with `npm i hyperspace -g`. Once installed, you'll have a `hyperspace` CLI command which can be used to start the service:
```
❯ hyperspace
Running hyperspace/3.9.1
Listening on /tmp/hyperspace.sock
```

Importantly, if you're an existing Hyperdrive daemon user, the Hyperspace server will read from your existing Hypercores (stored in `~/.hyperdrive`), whereas new users can find their cores in `~/.hyperspace`. This lets you run the old and the new on the same storage directory, keeping things compatible. In the future, launching Hyperspace will perform a migration, moving data from `~/.hyperdrive` to `~/.hyperspace`. Once this migration is done, your Hyperdrive daemon will no longer work (unless you manually move the data), so there's no looking back!

## The New and Improved Hyperdrive Service

While Hyperspace introduces major architectural changes to the stack, we've been careful to preserve the Hyperdrive daemon's functionality. Since FUSE and the CLI are only relevant for Hyperdrives, we've created a separate companion service for these components, [`@hyperspace/hyperdrive`](https://github.com/hyperspace-org/hyperdrive-service).

Being a companion service, `@hyperspace/hyperdrive` is designed to run alongside Hyperspace. If you're an existing user of the Hyperdrive daemon, you shouldn't notice any major changes. Installing the module globally will give you the same `hyperdrive` CLI as before.

If you've been using the `client.fuse` API in the `hyperdrive-daemon-client` module, that logic has been moved into the Hyperdrive service's client library:
```javascript
const fuseClient = require('@hyperspace/hyperdrive/client')
await fuseClient.info('/home/andrew/Hyperdrive/my-mount')
```

As part of this refactor, we've also gone ahead and added some flexibility on the FUSE side of things. Your root drive can now be mounted anywhere, not just at `~/Hyperdrive`, and we put together a handful of end-to-end FUSE tests to keep things reliable.

## Compatibility Additions to the Daemon Client

The daemon's gRPC API is now deprecated, but we didn't want to introduce significant breaking changes for users of the `hyperdrive-daemon-client` module. The 2.0 release of the client serves as a compatibility release -- it's a "drop-in" replacement for the 1.0 version, but is designed to work with Hyperspace instead.

There are a few small differences, though, hence the major bump. The `fuse` API has been moved into `@hyperspace/hyperdrive`, and the `peersockets` API just directly proxies to a [Peersockets 1.0](https://github.com/andrewosh/peersockets) instance. The 1.0 release of Peersockets has a couple of breaking changes as described in [UPGRADE.md](https://github.com/andrewosh/peersockets/blob/master/UPGRADE.md), which were introduced in order to be consistent with Hypercore extension usage.

If you're just getting started with the stack, we recommend using [`hyperdrive`](https://github.com/hypercore-protocol/hyperdrive) with a `RemoteCorestore` instead of the `hyperdrive-daemon-client`, as described above. We expect the 2.0 client release to tide folks over while they transition to using Hyperdrive directly, but the client is now considered "soft deprecated".

## Stay in Touch

Having tested Hyperspace for a few months now, we're very excited about this release. You might still hit a few bugs here or there, so you can reach us and join the discussion in the [Hypercore Protocol Discord](https://chat.hypercore-protocol.org). As always, feel free to open issues on the GitHub repos as well.

## Thanks

And thanks to the community members who have jumped in to fix bugs, add docs, and explore what Hyperspace can do. You definitely won't want to miss [Frando's Rust implementation](https://github.com/frando/hyperspace-rs), which paves the way for working with classic, Node-based Hypercores in Rust, and [vice versa](https://github.com/datrs).

\- Andrew and team
