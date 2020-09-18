---
title: Announcing Hyperdrive v10
description: Hyperdrive is now faster and more reliable. Plus it comes with a daemon.
date: 2020-05-14
first_author: Andrew Osheroff
first_author_page: https://twitter.com/andrewosh
tags:
  - hyperdrive
  - release
layout: layouts/post.njk
---
For the past year, we've been working hard on the v10 release of Hyperdrive. After a long period of beta testing, we're excited to announce that it's ready for general use!

Hyperdrive is a peer-to-peer filesystem that's designed to help you share files quickly and safely, directly from your computer. Hyperdrive v9, along with many other modules prefixed by 'hyper', has served as the backbone for [Dat](https://dat.foundation) for many years -- you might already be familiar with Hyperdrive if you've dug into Dat's internals.

Leading up to this release, we've done a bit of restructuring: Hyperdrive and its many hyper-siblings now live under a small, technically-focused brand/organization called the [Hypercore Protocol](https://hypercore-protocol.org). Practically, this change means very little beyond branding, but we're hoping it will give the modules a chance to shine on their own. In light of that, here's a look inside Hyperdrive.

In this post we'll step through some of the improvements we've made in v10, explain how Hyperdrive fits into the broader Hypercore Protocol ecosystem, and show you how to get started using it. This release is only the beginning, and we describe our next steps in the ["Looking Forward"](#looking-forward) section.

Here's the TL;DR:
1. __Improved Indexing__: We're using a new [HAMT](https://en.wikipedia.org/wiki/Hash_array_mapped_trie)-based indexing structure called a [Hypertrie](https://github.com/hypercore-protocol/hypertrie), which gives huge perf/scaling boosts.
2. __Mounts__: You can now "link" other peoples' drives into your own.
3. __Hyperdrive Daemon__: We've created a cross-platform daemon that provides both FUSE and gRPC access to daemon-managed drives.
4. __Better Foundations__: We've recently introduced the [Hyperswarm](https://hypercore-protocol.org/#hyperswarm) DHT, and improvements to the [Hypercore protocol](https://github.com/hypercore-protocol/hypercore-protocol), which have helped make our whole stack snappier and more reliable.

## What's Hyperdrive?

Hyperdrive is a POSIX-like filesystem implementation, written in Node.js, that's designed to be the storage layer for fast, scalable, and secure peer-to-peer applications. For most developers, working with a Hyperdrive should feel just like using Node's standard `fs` module, with only minor additions. Our main goal has always been to make it possible to share entire filesystems with others using a single 32-byte key (i.e. `hyper://ab13d...`). We'll refer to Hyperdrive filesystems as "drives" from here on out.

Drives are great for applications in which a single writer wants to distribute large, mutable collections of files to many readers. A file collection might be a video library, a personal blog, a scientific dataset, or what have you. Like BitTorrent, peers can download files from other peers without sacrificing trust (drive contents are signed by the original author).

Unlike BitTorrent, files can be added or modified after a drive is created, and peers can "watch" a drive for updates, meaning update notifications are dispatched to readers in __realtime__!

Importantly, drives support efficient random-access file reads, meaning that you can seek through a video and it will download only the portions of the video you're viewing, on-demand. We call this property "sparse downloading", and it's great for things like large websites (think all of Wikipedia mirrored to a drive) where readers only view single pages at a time.

Here's an example of sparsely downloading images from a large drive:
<div class="video-container-lg" id="van-gogh-vid">
  <video src="/video/van_gogh.mp4" autoplay="" loop="" muted="" playsinline=""></video>
</div>

Under the hood, Hyperdrive is built using two append-only log data structures called [Hypercores](https://github.com/mafintosh/hypercore), one for an efficient metadata index and one for binary file content. You can learn more about Hypercore from the [Hypercore Protocol website](https://hypercore-protocol.org). Hypercore gives us a fast and secure foundation for exchanging ordered blocks of data, but a good filesystem depends on a good index.

<div class="video-container">
  <video src="/video/trie.mp4" autoplay="" loop="" muted="" playsinline=""></video>
</div>

To support performant filesystem operations, such as directory traversals, we've layered a indexing data structure on top of Hypercore called a Hypertrie, which is an append-only implementation of a [hashed array-mapped trie](https://en.wikipedia.org/wiki/Hash_array_mapped_trie). Painting a complete picture of Hypertrie is a blog post in itself, but the most important takeaway is that it lets us locate file/directory metadata, which is potentially scattered across many peers, using `O(log_4(n))` network requests, worst case. In practice, we use a specialized Hypercore extension to make this dramatically faster (`O(1)` in most cases).

## Better Performance and Reliability

The Hypercore Protocol ecosystem has seen major improvements recently, and Hyperdrive directly benefits from all of them. Most significantly, Hyperdrive v10 leverages [Hypertrie](https://github.com/mafintosh/hypertrie) for __better indexing__, and [Hyperswarm](https://github.com/hyperswarm/hyperswarm) for __better networking__.

### Networking

Hyperswarm is a Kademlia-backed DHT implementation that's designed specifically for the home. It uses a distributed approach to holepunching -- peers in the DHT can help bootstrap connections to other peers -- allowing us to traverse through the vast majority of home routers.

Hyperswarm also comes full of heuristics designed to work around offline nodes and keep routing tables healthy. When combined with the holepunching, the heuristics make it dramatically faster to both discover and connect to peers.

### Indexing

Hyperdrive v9 used an indexing data structure that worked well for small drives, but quickly broke down as drives or directories grew large. In v10 we're using [Hypertrie](https://github.com/mafintosh/hypertrie). It scales nicely -- as a demo, we put a complete Wikipedia mirror (tens of millions of files, split across a few directories) into a drive, and reads remained very fast.

We'll expand on Hypertrie in a follow-up post, but for now the main takeaway is that your directories can be as large as you (realistically) like, and file lookups will stay snappy!

## What's New?

For this release we've focused on features that improve usability, simplify drive management, and reduce the friction of sharing. The two largest things we'd like to introduce are __mounts__, which let you create nested Hyperdrives, and the __Hyperdrive daemon__, for serving as a one-stop shop for managing collections of drives.

### Hyperdrive Mounts

Hypercore gives Hyperdrive many of its nice features, such as sparse downloading, for "free" -- the bulk of the work is handled at that layer. Hypercore, however, is fundamentally a single-writer data structure. A core's writer maintains a private key which is used to sign all appended data, making it possible for readers to exchange data amongst themselves without fear of tampering. Only letting one person (on one machine!) make changes to a drive is a big limitation, though. Not surprisingly, support for multiple writers has long been one of our most-requested features.

In v10, we don't go all the way to a general multi-writer solution; solving multi-writer scalably, without incurring major performance penalties or confusing UX, remains a research question for us. That said, v10 introduces mounts, which are pretty much "links" to other Hyperdrives that look and act like normal directories.

<div class="video-container">
  <video src="/video/metadata-and-content-and-mounts.mp4" autoplay="" loop="" muted="" playsinline=""></video>
</div>

Mounts open up lots of opportunities, both for more granular sharing and for fun multi-user applications. On the sharing side, you might create a `projects/` directory which contains mounts like `projects/my-module`, `projects/my-website` -- one drive for each thing you're working on. With mounts, you can share `my-website` on its own, without giving away access to everything in `projects/`. This pattern is especially handy in the daemon, which we'll talk about next.

Things get more interesting when the drives you're mounting aren't your own.  We've had a lot of success with a "groups" pattern, wherein a "group owner" first creates a top-level group drive, then mounts "user profiles" within the group:
```bash
/my-group  // Owned by the group owner (say User A)
  /user-a  // Owned by User A
  /user-b  // Owned by User B
  ...
```
Using this pattern, you can write simple "groupware" that aggregates content across users, using little more than a recursive `readdir` on the group drive. For example, to find all blog posts in the group, you might search for all Markdown files in each user's `blog/` directory.

#### Interested in using mounts now?

If you're using Hyperdrive directly, the `drive.mount(path, key, [opts])` method is what you're after. It works as you'd expect from the args (mount `key` at `path`), and the options can include a static drive version.

It's also easy to create mounts in the daemon through the CLI, which we'll describe next.

## The Hyperdrive Daemon

Hyperdrive is built with modular storage and networking in mind -- you can store drive contents however you like, and you can replicate them over any Node.js stream. This flexibility has benefits, but it makes it harder to get started. To that end, we've created a cross-platform background service (a daemon) that handles storage/networking for you, while giving you a variety of ways to access daemon-managed drives.

The daemon's a long-running service, so it can keep your drives online and available to readers. It's also great for DHT health: since the node on your computer is stable, the DHT's routing table contains fewer offline nodes. This translates to faster key lookups, meaning faster loading times.

Most importantly, the daemon serves as a central point for exposing drives to external services -- currently we support a [gRPC](https://grpc.io/) API, with a corresponding [Node.js client library](https://github.com/andrewosh/hyperdrive-daemon-client), and a FUSE interface.

[FUSE](https://github.com/libfuse/libfuse) allows us to emulate a native filesystem directory from within our Node.js code. It lets us turn Hyperdrives into normal directories on your computer! This means that whenever the daemon is running, you'll be able to access drives directly from within the OSX Finder, say, as virtual directories inside `~/Hyperdrive`.

With FUSE, drives are instantly accessible to other programs. You can watch movies using VLC, load PDFs using your favorite reader program, and use Unix utilities like `find` and `ls` to explore drives. We go into more depth in the "Getting Started" section below.

<div class="video-container-lg" id="tree-vid">
  <video src="/video/tree4.mp4" autoplay="" loop="" muted="" playsinline=""></video>
</div>

The `hyperdrive` CLI tool contains a handful of commands both for interacting with FUSE, and for displaying information about drives. It also provides `import` and `export` commands, for those users who don't want to mess around with `~/Hyperdrive`.

While the `~/Hyperdrive` directory was added to simplify end-user UX, the gRPC API exists for developers -- you can now program with Hyperdrive in any language, without needing to deal with tricky networking APIs. If you're using Node.js, the client library gives you a `RemoteHyperdrive` interface that feels exactly like a normal drive. To get started, jump [here](#With-the-Client-Library).

We're hoping that the daemon provides a frictionless entry point both for end-users looking to share data, and for developers who want to build apps and services using Hyperdrive. If you have thoughts or feedback on the UX, don't hesitate to drop into our chat!

## Getting Started

The best way to jump into the Hyperdrive stack is to install the daemon. This can be done through NPM:
```bash
❯ npm install hyperdrive-daemon -g
```
Once the daemon's installed, you'll have access to the `hyperdrive` CLI command. From here, you have a few options. It's best to consult the [daemon's README](https://github.com/andrewosh/hyperdrive-daemon) for a more detailed guide, but we'll highlight the key steps here.

### With FUSE

*Note: FUSE is currently only available on Linux and OSX. The CLI's `import` and `export` commands can be used to move data in/out of drives on Windows.*

Immediately after installation, you'll need to do a one-time setup step, which will request `sudo` access -- don't worry, the daemon itself does not run as root. This is necessary in order to configure the bundled FUSE kernel modules:
```bash
❯ hyperdrive fuse-setup
```

After this, start the daemon normally and you'll notice that your `~/Hyperdrive` directory exist and contains a subdirectory called `Network`. If you see `Network`, you're good to go.

As described in the walkthrough in the daemon README, `~/Hyperdrive` is your "root drive," which you can think of as a replacement for your Home directory. You can create subdirectories like Documents, Videos, or Projects in it, as you like.

`~/Hyperdrive/Network` is a "magic directory" in that it does not actually exist inside of your root drive. It's there to make it easy to:
1. Get storage/networking statistics as json files in `Network/Stats`
2. See which drives you're announcing to the swarm in `Network/Active`
3. Access any drive in the world by key at `Network/<drive-key>`

Details about `Network`, along with explanations of all the CLI commands you can use to populate and explore your root drive, can be found in the [README](https://github.com/andrewosh/hyperdrive-daemon#usage).

### With the Client Library

If you have a daemon instance running, you can use the [`hyperdrive-daemon-client`](https://github.com/andrewosh/hyperdrive-daemon-client) module to create `RemoteHyperdrive` objects. Under the hood, these will send commands over gRPC to daemon-managed drives.

The `RemoteHyperdrive` API mirrors Hyperdrive's. The following code snippet will create a client instance, use that client to create a new drive, then write a file to the drive:
```javascript
const { HyperdriveClient } = require('hyperdrive-daemon-client')

// Auto-connects to the daemon
const client = new HyperdriveClient()
await client.ready()

// Creates a new drive and writes a file
const drive = await client.drive.get()
await drive.writeFile('foo.txt', 'bar')
```

The daemon's README gives more examples. As of today, we only have a Node.js client, but the daemon's [gRPC schemas](https://github.com/andrewosh/hyperdrive-schemas/tree/master/schemas/daemon) are available, and we'd welcome any efforts to create clients in other languages.

### With Beaker
The [Beaker Browser](https://beakerbrowser.com) makes heavy use of Hyperdrive internally. Beaker 1.0 Beta, which is also being released *today*, actually installs and manages the daemon for you in the background! Beaker comes packed with authoring tools for creating P2P websites and sharing them with others.

The [Beaker developer portal](https://beaker.dev) contains thorough docs and tutorials (Tip: open that site in Beaker to make the tutorials interactive!), so you'll be off the ground immediately, building Hyperdrives containing fully-featured web applications (think personal wikis, photo albums, blog aggregators, and more).

*You can find this blog post in Beaker at [hyper://e36050c2651b2bd0bd265fc311c122122a5490bc11a17606201964246b69e4ef](hyper://e36050c2651b2bd0bd265fc311c122122a5490bc11a17606201964246b69e4ef)!*

### Standalone
You'll probably want to use the daemon/client most of the time, but in case you don't - perhaps in a one-off script or some kind of embedded scenario - you can use Hyperdrive as a module inside your program. The [README](https://github.com/hypercore-protocol/hyperdrive) shows you how, and it's also where you'll find complete API docs.

[Here's a Gist](https://gist.github.com/andrewosh/1f3ae698ba42f7a382a5b85ab5305b88) containing a small, end-to-end example showing how to use Hyperswarm to discover and sync a Hyperdrive from another peer.

In the future, we plan on making a few detailed tutorials about programmatic Hyperdrive usage. Remember, if you're looking for the simplest solution, check out the daemon and/or Beaker!

## Looking Forward

As you can see, a lot's been happening lately! But far more remains to be done. With these features released, here's a sketch of what we plan on tackling next.

### More Indexing Improvements

The current implementation of Hypertrie's led to major gains. That said, it lacks a few features you might expect from a filesystem, most importantly atomic renames. Also, due to the way trie iteration works, both symlinks and mounts are slightly less efficient than they need to be.

We've been sitting on a new, heavily-fuzzed trie implementation for a while now, but we haven't had the cycles to integrate it yet. It supports symlinks (directly, as opposed to in the Hyperdrive layer), mounts, and atomic renames through a unified iterator abstraction that we're calling a "trie controller". It's better all around.

And don't worry, the new trie will be fully backwards-compat with the one we're releasing now.

### Deduplication

The daemon affords us many new opportunities by virtue of its both storing all drives in one location on disk, and handling networking in one place. By having total control of storage/networking, we can perform optimizations across your entire drive collection.

On the networking side, we're investigating methods for deduplicating block requests that have already been satisfied by other drives. The end-goal here is: "you should never have to download the same data twice, even if its contained in different drives".

On the storage side, we're considering supporting content-addressed block storage, meaning if you have two similar drives, only the set of unique blocks will be persisted on disk -- common blocks will only be stored once.

### Random-Access Writes

While a writer can update their drives however they like (i.e. adding new files, deleting files, appending to files, etc.), certain operations are more efficient than others. Unfortunately, editing existing files is one of the inefficient ones -- it currently results in file duplication. This is bad news if you want to run a database on FUSE or append to log files.

There are two ways to remedy this:
1. Better garbage collection, where old versions of files can be cleared from disk easily (fine for simple cases, but still bad for databases).
2. Efficient file updates, where writing to an existing file does not lead to data copying.

We're exploring various approaches based on [inodes](https://en.wikipedia.org/wiki/Inode) for (2). As with general multiwriter, there's a tricky balancing act involved. An index that supports these updates will unavoidably increase read latencies for files that don't have random-access modifications. This won't cut it if you're watching a large movie, never modified after it was first written, from start to end.

We'll surely have to use a hybrid approach; we're still actively researching this.

### Garbage Collection

As described above, the append-only nature of file updates means that we're not exactly being conservative with your disk space. Hyperdrive does not currently support clearing old file versions, but we've taken a step in that direction with "tags."

Using tags, you can assign names to drive versions you'd like to keep around. We'll shortly be adding support for a `clearUntagged` method that will remove untagged file versions from disk.

With `clearUntagged`, random-access writes become less essential, so we're hoping it will serve as a nice near-term stopgap.

### Union Mounts

Mounts currently cannot overlap with one another. The group model works around this limitation, enabling useful multi-user applications, but there are many cases where you'd like to display a "merged view" over many drives.

To accomplish this, we're thinking about ways to extend mounts. The "trie controller" design we alluded to above makes experimentation easier here. A simple union mounts feature, without any opinionated conflict resolution (i.e. displaying conflicting files side-by-side), is a natural next step.

A general multiwriter solution, with customizable hooks for conflict resolution, remains on our to-do list, but that's still a far-future feature. We want to see how far we can go with mounts first.

## Many Thanks

Gearing up for this release has been a big group effort over the past year, and we owe a lot to the many contributors in the Dat community who have helped test alpha versions of the stack, fix bugs, and review docs.

A very special thanks goes to Samsung, whose generous [Samsung Next](https://samsungnext.com/) grant funded a massive chunk of this work. Huge thanks to Ricardo and the rest of the Samsung Next team!

## Learning More
To learn more about how Hyperdrive works under the hood, your best bet is to check out the source code on GitHub. Our code's currently split across a number of repos, most within the [Hypercore Protocol](https://github.com/hypercore-protocol) organization. Here are a few direct links to the stuff we've discussed in this post:
* [Hyperdrive](https://github.com/hypercore-protocol/hyperdrive)
* [Hyperdrive Daemon](https://github.com/hypercore-protocol/hyperdrive-daemon)
* [Hypercore](https://github.com/hypercore-protocol/hypercore)
* [Hyperswarm](https://github.com/hyperswarm/hyperswarm)
* [Hypertrie](https://github.com/hypercore-protocol/hypertrie)

## Chat with us
If you have questions about Hyperdrive's design, or want to talk about the stack, message us on [Discord](https://chat.hypercore-protocol.org) -- that's where we'll be having development-oriented conversations.

If you run into any bugs, go ahead and open issues on our [Community repo](https://github.com/hypercore-protocol/community).

And also feel free to send DMs to anyone on the team:
* [@mafintosh](https://twitter.com/mafintosh)
* [@pfrazee](https://twitter.com/pfrazee)
* [@andrewosh](https://twitter.com/andrewosh)

## Credits
Big thanks to [@mafintosh](https://twitter.com/mafintosh) for the visualizations and [@pfrazee](https://twitter.com/pfrazee) for proofreading.

In our demo videos, we used content from The Internet Archive's [van Gogh collection](https://archive.org/details/academictorrents_c8b687c984d3d902310f27d56759ed69f5e1b4a7).
