---
title: Overview for Developers
---

![earthstar logo](/img/earthstar-logo-small.png)

Earthstar is a distributed p2p database and app-making toolkit.

You can use it to build Slack or Discord style apps for small communities, but more than just chat -- any kind of collaborative app (todo lists, wikis, games...).

It's sort of a [delta-state CRDT](https://arxiv.org/pdf/1603.01529.pdf) for low-trust environments, implementing a last-writer-wins key-value store.

Any two Earthstar peers can sync with each other if they can make a network connection.  Right now we're using "pub servers" with public IP addresses which act as supernodes so that apps can work in regular browsers.  We also plan to implment direct p2p connections but that will not work in regular browsers because of NATs and firewalls.

## Comparison with other distributed tools

Earthstar is sort of halfway between Scuttlebutt and CouchDB.

|                             | Scuttlebutt                 | Earthstar              | CouchDB              |
|-----------------------------|-----------------------------|------------------------|----------------------|
| data model                  | append-only log of messages | key-value database     | key-value database   |
| sync method                 | just get logs in order      | set replication        | ?                    |
| network                     | one global network          | small closed networks  | small closed networks |
| authors are...              | ✅ identified by pubkey      | ✅ identified by pubkey | 🚫 not identified     |
| messages are                | ✅ signed                    | ✅ signed               | 🚫 not signed         |
| safe with untrusted peers   | ✅ yes                       | ✅ yes                  | 🚫 no                 |
| hash backlinks              | ✅ yes                       | 🚫 no, data is mutable  | 🚫 no                 |
| proof of complete feed      | ✅ yes                       | 🚫 no                   | 🚫 no                 |
| mutability, deletion        | 🚫 no                        | ✅ yes                  | ✅ yes                |
| immutability                | ✅ yes                       | ⏳ planned              | 🚫 no                 |
| multiple devices per author | 🚫 no                        | ✅ yes                  | ✅ yes                |
| several authors can edit the same data| 🚫 no              | ✅ yes                  | ✅ yes                |
| partial replication         | 🚫 not really                | ✅ yes                  | ✅ yes                |
| multiple redundant pubs     | ✅ yes                       | ✅ yes                  | ✅ yes              |
| conflict resolution         | N/A, immutable              | 🐛 simpler              | ✅ sophisticated      |

## Goals

* Design for small closed groups of 1-100 people, not giant global networks
* Provide building blocks for user safety and usability: access control, deletion, mutability, multiple devices per author, multiple authors can edit the same data, partial replication, encryption
* Accessible to users
  * Privacy model should be intuitive for users to understand.
  * Works in normal browsers
  * Reasonable device requirements (storage, bandwidth, CPU)
  * Offline-first, does not require reliable internet connection
* Accessible to app developers
  * Lots of documentation
  * It's easy to make small apps
* Accessible to developers of Earthstar itself
  * Choose simplicity of code and algorithms instead of more sophisticated, more efficient features
    * We don't have to scale very big!
  * Use boring technology and common paradigms like HTTP and JSON
* Ecological
  * No blockchain energy-wasting nonsense
* DIY, non-corporate
  * Made by volunteers
  * You own and run your own infrastructure

## Progress so far

* ✅ Standardized message format [(Specification)](../reference/earthstar-specification)
* ✅ A [reference implementation, in Typescript](https://www.npmjs.com/package/earthstar)
    * Storage: ✅ in-memory, ✅ sqlite, ⏳ indexeddb,
    * ✅ Message validation.  This is modular and can support multiple formats at once.
    * ✅ Pub-style HTTP server
    * ✅ Command line helper tool
    * A sync algorithm (✅ working, ⏳ effecient)
* ✅ A couple of little demo apps

## How does it work?

* An Earthstar database holds key-value pairs, which we call "paths" and "documents".
* Put these databases all over the place -- in the browser, in cloud servers, in native apps
* They all sync with each other over HTTP or duplex streams
* They can do partial sync, drop older data, etc.

Users are identified with public keys, just like SSB.

## Networking

Peers find each other for syncing in a variety of ways: by talking to cloud servers (like SSB pubs), or over hyperswarm or libp2p, bluetooth, trading USB drives, whatever you like.  This part is not as standardized.

## Groups and communities

Data and users are grouped into independent `workspaces` (like a Slack workspace, or SSB's scuttleverses, or separate DATs).  Earthstar is designed for both small closed workspaces where you want all the data, but it can also work in large open workspaces where you only want data from people you follow.  It doesn't have a concept of "following" yet but it would be easy to add on top.

## Security and data guarantees

* ✅ Each message is signed by the author
* ⏳ End-to-end encryption is coming soon

Untrusted peers can help replicate data without modifying it.  The worst they can do is withhold specific documents without your knowlege.  If there are some trusted peers around, they will fill in the gaps for you.

Earthstar does not guarantee causal order or help you prove you have a complete set of messages.  You can add some of those things on top, at the application level.

## Write permissions

Write permissions are encoded into each document's path using a tilde `~` as a marker.  Documents can be writable by anyone, or write-limited to specific users.

Example paths:

* `/wiki/kittens` - anyone can write here
* `/~@aaaa.xxxx/about` - only user @aaaa can write here
* `/~@aaaa.xxxx/follows/@bbbb.xxxx` - only @aaaa can write here
* `/whiteboard/~@aaaa.xxxx~@bbbb.xxxx` - both @aaaa and @bbbb can write here, and nobody else can

## Data mutability and conflicts

Documents can be overwritten and deleted.  Deleted data is lost -- there's no append-only log.

Conflicts may occur between different document versions that have the same path.  The winning version is the one with the highest timestamp (as self-reported by the author, with some safeguards against future timestamps).

Within one path we keep the newest document from each author, even if it's not the overall winner, to allow manual conflict resolution later.

Earthstar is not designed for fancy conflict resolution.  It's best to design apps so they don't need it -- for example, let people add comments to something as separate documents, instead of everyone editing the same document.

⏳ We plan to also support immutable documents that nobody can edit after they're published.

## Indexing

The core Earthstar library provides indexing and querying capability so apps don't have to implement it themselves.  Earthstar documents are like independent rows in a database, not like accumulating patches in a log, so our indexing is not about maintaining an accumulated state but just about searching for documents.

The main way to query for documents is by path, or path prefix, but you can also search by author and timestamp.

## Selective sync and sparse mode

Apps can hold any subset of the documents.  For example, they could sync only recent documents whose paths begin with `/chess/`, and ignore all other documents.  This makes initial sync faster and saves bandwidth and storage space.

 ⏳ Apps will be able to sync document metadata without getting the actual document content.  This is helpful if the content is large, such as images, and we only want to fetch the content when it's viewed.
