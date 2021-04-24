---
title: Earthstar Specification
---

Format: `es.4`

Document version: 2021-04-23.0

> The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL
> NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED",  "MAY", and
> "OPTIONAL" in this document are to be interpreted as described in
> [RFC 2119](https://tools.ietf.org/html/rfc2119).
> "WILL" means the same as "SHALL".

## Libraries Needed to Implement Earthstar

To make your own Earthstar library, you'll need:

### ed25519 Signatures

This is the same cryptography format used by Secure Scuttlebutt and DAT/hyper.

### base32 Encoding

Almost anywhere that binary data needs to be encoded in Earthstar, it's done with base32: public and private keys, signatures, hashes.  The exception is binary document content which is base64 (see next section).

We use [RFC4648](https://tools.ietf.org/html/rfc4648#section-6) with lowercase letters and no padding.  The character set is `abcdefghijklmnopqrstuvwxyz234567`.

Our encodings are always prefixed with an extra `b` character, following the [multibase](https://github.com/multiformats/multibase) standard.  The `b` format is the only format supported in Earthstar.  Libraries MUST enforce that encoded strings start with `b`, and MUST NOT allow any other encoding formats.

Libraries MUST be strict when encoding and decoding -- only allow lowercase characters; don't allow a `1` to be treated as an `i`.

> **Why this encoding?**
>
> * We want to use encoded data in URL locations, which can't contain upper-case characters, so base64 and base58 won't work.
> * base32 is shorter than base16 (hex).
> * The choice of a specific base32 variant was arbitrary and was influenced by the ones available in the multibase standard, which is widely implemented.
> * The leading `b` character serves two purposes: it defines which base32 format we're using, and it prevents encoded strings from starting with a digit.  This makes it possible to use encoded strings as standards-complient URL locations, as in `earthstar://gardening.bajfoqa3joia3jao2df`.

### base64 Encoding

Document contents may only contain utf-8 strings, not arbitrary binary bytes.  Applications wishing to store binary data SHOULD encode it as base64.

The recommended format is [RFC4648](https://tools.ietf.org/html/rfc4648#section-4) base64, regular (not-URLsafe), with padding.  This is the same format used by `atob()` and `btoa()` in Javascript.

> **Why?**
>
> * This encoding is built into web browsers and probably implemented by native code, making it faster for large amounts of data and easier to integrate with other web APIs such as canvas, IndexedDb, displaying images, etc.
> * It's more widely used than base32.
> * It uses less space than base32, which matters more because this will be used for large files.
> * We don't have the same character set restrictions for this use case that we do for the base32 use cases (paths, author addresses, etc).

### Indexed Storage

Earthstar messages are typically queried in a variety of ways.  This is easiest to implement using a database like SQLite, but if you manage your own indexes you can also use a key-value database like leveldb.

## Vocabulary and concepts

**Library, Earthstar library** -- In this context, an implementation of Earthstar itself.

**App** -- Software which uses Earthstar to store data.

**Document** -- The unit of data storage in Earthstar, similar to a document in a NoSQL database.  A document has metadata fields (author, timestamp, etc) and a **content** field.

**Path** -- Similar to a key in leveldb or a path in a filesystem, each document is stored at a specific path.

**Author** -- A person who writes documents to a workspace.  Authors are identified by an ed25519 public key in a format called an **author address**.  It's safe for an author to use the same identity from multiple devices simultaneously.

**Workspace** -- A collection of documents.  Workspaces are identified by a **workspace address**.  Workspaces are separate, unrelated worlds of data.  Each document exists within exactly one workspace.

**Format, Validator** -- The Earthstar document specification is versioned.  Each version of the specification is called a document **format**, and the code that handles that format is called a **Validator**.

**Peer** -- A device which holds Earthstar data and wishes to sync with other peers.  Peers may be individual users' devices and/or pub servers.  A peer may hold data from multiple workspaces.

**Pub server** -- (short for "public server") -- A peer whose purpose is to provide uptime and connectivity for many users.  Usually these are cloud servers with publically routable IP addresses.

## Data model

```
// Simplified example of data stored in a workspace

Workspace: "+gardening.friends"
  Path: "/wiki/shared/Flowers"
    Documents in this path:
      { author: @suzy.b..., timestamp: 1500094, content: 'pretty' }
      { author: @matt.b..., timestamp: 1500073, content: 'nice petals' }
      { author: @fern.b..., timestamp: 1500012, content: 'smell good' }
  Path: "/wiki/shared/Bugs"
    Documents in this path:
      { author: @suzy.b..., timestamp: 1503333, content: 'wiggly' }
```

A peer MAY hold data from many workspaces.  Each workspace's data is treated independently.  Each document within a workspace is also independent; they don't form a chain or feed; they don't have Merkle backlinks.

A workspace's data is a collection of documents by various authors.  A peer MAY hold some or all of the documents from a workspace, in any combination.  Apps MUST assume that any combination of docs may be missing.

Each document in a workspace exists at a path.  For each path, Earthstar keeps the newest document from each author who has ever written to that path.

In this example, the `/wiki/shared/Flowers` path contains 3 documents, because 3 different authors have written there.  They may have written there hundreds of times, but we only keep the newest document from each author, in that path.

"Newest" is determined by comparing the `timestamp` field in the documents.  See the next section for details about trusting timestamps.

When looking up a path to retrieve a document, the newest document is returned by default.  Apps can also query for the full set of document versions at a path; the older ones are called **history documents**.

### Ingesting Documents

When a new document arrives and an existing one is already there (from the same author and same path), the new document overwrites the old one.  Earthstar libraries MUST actually delete the older, overwritten document.  The author's intent is to remove the old data.

The process of validating and potentially saving an incoming document is called **ingesting**, and it MUST happen to newly obtained documents, whether they come from other peers or are made as local writes.  Earthstar libraries MUST use this ingestion process:

```ts
// Pseudocode

Ingest(newDoc):
    // Check doc validity -- bad data types, bad signature,
    // expired ephemeral doc, wrong format string,
    // timestamp too far in the future, ...
    if !isValid(newDoc):
        return "rejected an invalid doc"

    // Check if it's obsolete
    // (Do we have a newer doc with same path and same author?)
    let existingDoc = query({author: newDoc.author, path: newDoc.path});
    if existingDoc exists && existingDoc.timestamp >= newDoc.timestamp;
        return "ignored an obsolete doc"

    // Overwrite older doc with same path and same author
    if existingDoc exists:
        remove(existingDoc)
    save(newDoc)
    return "accepted a doc"
```

Each document has a `format` field which specifies which data [Format](#format) it is.  The `isValid` function in this pseudocode represents a call to the [Format Validator](#format) which is responsible for enforcing the rules of that format.

### Timestamps and Clock Skew

TODO: discuss timestamps

For now, see the [Timestamp](#timestamp) section, lower in this document, for rules about handling clock skew and documents from the future.

Also see the (non-normative) document [How does Earthstar handle timestamps, and can it recover from a device with a very inaccurate clock?](https://github.com/earthstar-project/earthstar/blob/main/docs/timestamps.md)

## Identities, Authors, Workspaces

### Character Set Definitions

```
ALPHA_LOWER = any of "abcdefghijklmnopqrstuvwxyz"
ALPHA_UPPER = any of "ABCDEFGHIJKLMNOPQRSTUVWXYZ"
DIGIT = any of "0123456789"

B32_CHAR = ALPHA_LOWER + any of "234567"
ALPHA_LOWER_OR_DIGIT = ALPHA_LOWER + DIGIT

ASCII = decimal character code 0 to 127 inclusive
      = hex character code 0x00 to 0x7F inclusive
        (no "extended ASCII" > 0x7F)

PRINTABLE_ASCII = characters " " to "~", inclusive
                = decimal character code 32 to 126 inclusive
                = hex character code 0x20 to 0x7E inclusive
```

In this document, everywhere we say ASCII we mean "standard ASCII and not extended ASCII".  Only characters <= `0x7f`, none higher.

### Workspace Addresses

```
WORKSPACE_ADDRESS = "+" NAME "." SUFFIX
NAME = one ALPHA_LOWER followed by 0 to 14 ALPHA_LOWER_OR_DIGIT
SUFFIX = one ALPHA_LOWER followed by 0 to 52 ALPHA_LOWER_OR_DIGIT
```

Example:
```
+gardening.jfao38ifjhaolie
```

A workspace address is a sequence of: plus character `+`, **name**, period `.`, **suffix**.

It MUST have those four elements in that order.

The **name**:
* MUST be 1 to 15 characters long, inclusive.
* MUST only contain digits `0-9` and lowercase ASCII letters `a-z`
* MUST NOT start with a digit

The **suffix**:
* MUST be 1 to 53 characters long, inclusive.
* MUST only contain digits `0-9` and lowercase ASCII letters `a-z`
* MUST NOT start with a digit

No uppercase letters are allowed.

Valid examples:

```
+a.b
+gardening.friends
+gardening.j230d9qjd0q09of4j
+gardening.bnkksi5na3j7ifl5lmvxiyishmoybmu3khlvboxx6ighjv6crya5a
+bestbooks2019.o049fjafo09jaf
```

Invalid examples:

```
+a.b.c       // too many periods
+80smusic.x  // name can't start with a digit
+a.4ever     // suffix can't start with a digit
+PARTY.TIME  // no uppercase letters
```

> **Why these rules?**
>
> These rules allow workspace addresses to be used as the location part of regular URLs, after removing the `+`.

Workspace suffixes may be used in a variety of ways:

* meaningful words similar to domain name TLDs
* random strings that make the workspace hard to guess
* public keys in base32 format, starting with `b`, to be used as the **workspace key** in future versions of Earthstar (see below).

Note that anyone can write to and read from a workspace if they know its full workspace address, so it's important to keep workspace addresses secret if you want to limit their audience.

#### Invite-only Workspaces (coming soon)

(This is also discussed in the Future Directions section: [Invite-only Workspaces](#invite-only-workspaces) )

In the future, Earthstar will support **invite-only** workspaces.

These separate "read permission" from "read and write permission".  This enables use-cases such as
* Workspaces where a few authors publish content to a wide audience of readers (who otherwise would be able to write to the workspace once they know its address)
* Pub servers which host workspaces without the ability to write their own docs into the workspace

Invite-only workspaces will have a **workspace key** and **workspace secret**.  The key is used as the workspace suffix, and the secret is given out-of-band to authors who should be able to write.

Only authors who know the workspace key can write to an invite-only workspace.  They will sign their documents with the workspace secret (in a new field, `workspaceSignature`, in addition to the regular author signature).

This will limit who can write, but anyone knowing the workspace address can still read.  To limit readers, authors may choose to encrypt their document content using the workspace key so that anyone with the workspace secret can decrypt it.

TODO: We need a way to tell if a workspace is invite-only or not just by looking at its address.  Perhaps it's invite-only if the suffix looks like a base32 pubkey (53 chars starting with 'b').

### Author Addresses

```
AUTHOR_ADDRESS = "@" SHORTNAME "." B32_PUBKEY
SHORTNAME = one ALPHA_LOWER followed by three ALPHA_LOWER_OR_DIGIT
B32_PUBKEY = "b" followed by 52 B32_CHAR

AUTHOR_SECRET = "b" followed by 52 B32_CHAR
```

Examples
```
address: @suzy.bo5sotcncvkr7p4c3lnexxpb4hjqi5tcxcov5b4irbnnz2teoifua
secret: becvcwa5dp6kbmjvjs26pe76xxbgjn3yw4cqzl42jqjujob7mk4xq 

address: @js80.bnkivt7pdzydgjagu4ooltwmhyoolgidv6iqrnlh5dc7duiuywbfq
secret: b4p3qioleiepi5a6iaalf6pm3qhgapkftxnxcszjwa352qr6gempa
```

An author address starts with `@` and combines a **shortname** with a **public key**.

* **Shortnames** are chosen by users when creating an author identity.  They cannot be changed later.  They are exactly 4 lowercase ASCII letters or digits, and cannot start with a digit.

* **Public keys** are 32-byte ed25519 public keys (just the integer portion, no wrapper or surrounding data structures), encoded as base32 with an extra leading "b".  This results in 52 characters of base32 plus the "b", for a total of 53 characters.

* **Private keys** (called "secrets") are also 32 bytes of binary data (just the secret integer), encoded as base32 in the same way as the public key.

Apps MUST treat authors as separate and distinct when their addresses differ, even if only the shortname is different and the pubkeys are the same.

Note that authors also have Unicode **display names** stored in their **profile documents**, and those can be changed and allow more freedom of expression.  See the next section.

### FAQ: Author Shortnames

> **Why shortnames?**
>
> Impersonation is a difficult problem in distributed social networks where account identifiers [can't be both unique and memorable](https://en.wikipedia.org/wiki/Zooko%27s_triangle).  Users have to vigilantly check for imposters.  Typically apps will treat following relationships as trust signals, displaying the accounts of people you follow in a different way to help you avoid imposters.
>
> Shortnames make user identifiers "somewhat memorable" to defend against impersonation.
>
> For example: In Scuttlebutt, users are identified by a bare public key and their display names are mutable.
>
> A user could create an account with a display name of "Cat Pictures" and get many followers.  They could then change the display name to match another user that they wish to impersonate.  Anyone who previously followed "Cat Pictures" is still following the account under the new name, causing the account to appear trustworthy in the app's UI.  Users decided to trust the account in one context (to provide cat pictures) but after trust was granted, the account changed context (to impersonate a friend).
>
> For example, let's say an app shows "✅" when you're following an account.  "✅ Cat Pictures @3hj29dhj..." renames itself to "✅ Samantha @3hj29dhj...", which is hard to tell apart from your actual friend "✅ Samantha @9c2j392hx...".
>
> Adding an immutable shortname to the author address makes this attack more difficult.  Users can now notice when display name is different than expected.
>
> For example "✅ Cat Pictures @cats.3hj29dhj..." renames itself to "✅ Samantha @cats.3hj29dhj...", which is easier to tell apart from your actual friend "✅ Samantha @samm.9c2j392hx...".
>
> Of course the attacker could choose to start off as "✅ Cat Pictures @samm.3hj29dhj...".  Users are expected to notice this as a suspicious situation when following an account.

> **Why are shortnames 4 characters?**
>
> Shortnames need to be long enough that they can express a clear relationship to the real identity of the account.
>
> They need to be short enough for users to intuitively understand that they are non-unique.

> **Why limit shortnames to ASCII?**
>
> Users would be better served if they could use their native language in shortnames, but this creates potential vulnerabilities from Unicode normalization.
>
> This usability shortfall is limited because shortnames don't need to be very expressive; users can use Unicode in the display name in their profile.

> **What if users want to change their shortnames?**
>
> Users can change their display names freely but their shortnames are fixed.  Modifying the shortname effectively creates a new identity and the user's followers will not automatically follow the new identity.
>
> Humane software must allows users to change their names.  (See [Falsehoods programmers believe about names](https://www.kalzumeus.com/2010/06/17/falsehoods-programmers-believe-about-names/)).  Choosing and changing your own name is a basic human right.
>
> Software should also help users avoid impersonation attacks, a common harassment technique which can be quite destructive.  Earthstar attempts to find a reasonable trade-off between these competing needs in the difficult context of a distributed system with no central name authority.
>
> Users who anticipate name changes, or dislike the permanence of shortnames, can choose shortnames which are memorable but non-meaningful, like `zzzz` or `oooo`.

> **Can users create two identities with the same pubkey but different shortnames?**
>
> Yes.  They are considered two distinct identities, although you can infer that they belong to the same person.

### Author Display Names and Profile Info

An author can have a **profile** containing their **display name**, biographic information, etc.  Profile data is stored in the `content` of a variety of documents under `/about/`:

```
displayNamePath = "/about/~" + authorAddress + "/displayName.txt"

Example:
/about/~@suzy.bo5sotcncvkr7p4c3lnexxpb4hjqi5tcxcov5b4irbnnz2teoifua/displayName.txt
```

Display names stored in profile information can be changed frequently and can contain Unicode.

The expected paths and format of the profile documents are described in our wiki at [Standard paths and data formats used by apps](https://github.com/earthstar-project/earthstar/wiki/Standard-paths-and-data-formats-used-by-apps#about-author-profile-info).  They are not part of this lower-level specification.

We may add more standard pieces of profile information later, such as following and blocking of other users, a paragraph about yourself, a user icon, etc, but this is not standardized yet.

However, apps SHOULD consider the `/about/` namespace to be a standardizable area and be extra thoughtful about what they write there.

> **Why "about"?**
>
> Secure Scuttlebutt uses "about" messages to describe people's profile information, and we've adopted that vocabulary.
>
> Also, "about" comes towards the beginning of the alphabet, so if peers sync their documents in alphabetical order by path (which may or may not happen), the `/about/` data will be some of the first data synced.

## Paths and Write Permissions

### Paths

Similar to a key in leveldb or a path in a filesystem, each document is stored at a specific **path**.

Rules:

```
// note that double quote is not included,
// it's just part of our notation in this specification
PATH_PUNCTUATION = any of "/'()-._~!$&+,:=@%"

PATH_CHARACTER = ALPHA_LOWER + ALPHA_UPPER + DIGIT + PATH_PUNCTUATION

PATH_SEGMENT = "/" + one or more PATH_CHARACTER
PATH = one or more PATH_SEGMENT
```

* A path MUST be between 2 and 512 characters long (inclusive).
* A path MUST begin with a `/`
* A path MUST NOT end with a `/`
* A path MUST NOT begin with `/@`, but it may contain `/@` in the middle.
* A path MUST NOT contain `//` (because each `PATH_SEGMENT` must have at least one `PATH_CHARACTER`)
* Paths are case sensitive.
* Paths MAY contain upper and/or or lower case ASCII letters plus the punctuation and numbers described above.
* Paths MUST NOT contain any characters except those listed above.  To include other characters such as spaces, double quotes, emojis, or other non-ASCII characters, apps SHOULD use [URL-style percent-encoding as defined in RFC3986](https://tools.ietf.org/html/rfc3986#section-2.1).  First encode the string as utf-8, then percent-encode the utf-8 bytes.
* A path MUST contain one or more `!` characters, anywhere, IF AND ONLY IF the document is ephemeral (because `deleteAfter` is non-null).  See the section on [Ephemeral Documents](#ephemeral-documents-deleteafter).

In the following examples, `...` is used to shorten author addresses for easier reading.
`...` is not actually related to the Path specification.
```
Example paths

Valid:
    /todos/123.json
    /wiki/shared/Dolphins.md
    /wiki/shared/Dolphin%20Sounds.md
    /about/~@suzy.bo5sotcn...fua/profile.json
    /wall/@suzy.bo5sotcn...fua/post123.md

Invalid: path segment must have one or more path characters
    /

Invalid: missing leading slash
    todos/123.json

Invalid: starts with "/@"
    /@suzy.bo5sotcn...fua/profile.json
```

> **Why these specific punctuation characters?**
>
> Earthstar paths are designed to work well in the path portion of a regular web URL.

> **Why can't a path start with `/@`?**
>
> When building web URLs out of Earthstar pieces, we may want to use formats like this:
>
> ```
> https://mypub.com/WORKSPACE/PATH_OR_AUTHOR
>
> https://mypub.com/+gardening.friends/wiki/Dolphins
> https://mypub.com/+gardening.friends/@suzy.bo5sotcncvkr7...  (etc)
> ```
> 
> The restriction on `/@` allows us to tell paths and author addresses apart in this setting.  It also encourages app authors to put their data in a more organized top-level prefix such as `/wiki/` instead of putting each author at the root of the path.
>
> Another solution was to use a double slash `//` to begin paths and avoid confusion with authors:
>
> ```
> Don't do this:
> https://mypub.com/+gardening.friends//wiki/Dolphins
>                                      ^
> ```
>
> ...but some webservers treat this as user error and rewrite the double slash to a single slash.  So we have to carefully avoid the double slash when building URLs.

### Path Characters With Special Meaning

* `/` - starts paths; separates path segments
* `!` - used if and only if the document is ephemeral
* `~` - (tilde) defines author write permissions
* `%` - for percent-encoding other characters
* `+@.` - used in workspace and author addresses but allowed elsewhere too

### Disallowed Path Characters

The list of ALLOWED characters up above is canonical and exhaustive.  This list of disallowed characters is provided only for convenience and is non-normative if it accidentally conflicts with the allowed list.

See the source code `src/util/characters.ts` for longer notes.

Character - reason for being disallowed

* space           - not allowed in URLs
* ASCII whitespace (tab, etc) - not allowed in URLs
* ASCII control characters (bell, etc) - not allowed in URL, and not visible
* `<>"[\]^{|}`    - not allowed in URLs.  `{}` MAY be used for path templates (see below)
* `*`             - MAY be used for glob-style querying (see below)
* \` backtick     - not allowed in URLs
* `?`             - to avoid confusion with URL query parameters
* `#`             - to avoid confusion with URL anchors
* `;`             - useful for separating several paths while still being legal in URLs
* non-ASCII chars - (above `0x7F`) to avoid trouble with Unicode normalization and canonicalization for signatures, and phishing attacks

> **Path templates and glob-style querying**
>
> Earthstar libraries MAY offer extra ways of querying paths that use the `{}*?` characters.  This is not standardized, but those characters are available because they're not allowed in normal paths.
>
> Example: you might be able to query for `/blog/v1/{category}/{postId}.json` and get back matching documents with the `category` and `postId` extracted into variables for you, similar to the way URL routes are specified in libraries like Express.
>
> Example: you might be able to do "glob-style" queries like `/blog/v1/**/*.json`.

> **Side note: The ASCII range of allowed path characters**
> 
> When handling path strings, you may find yourself needing to choose a separator character that will lexicographically sort before or after all allowed paths.
>
> If you're handling entire paths, this is easy, because all legal paths start with `/`.
>
> If you're handling path segments (the parts between slashes), the range is wider:
>
> Amongst the allowed path characters, the lowest ASCII value is exclamation mark `!` and the highest ASCII value is tilde `~`.  (Of course, not all ASCII values between those extremes are allowed.)
> 
> Therefore if you need an ASCII value that's lower than any possible path segment, anything less than or equal to space (`0x20`, decimal 32) will do.  And the only ASCII value higher than all path characters is `DEL` (`0x7F`, decimal 127).  Only standard ASCII values are allowed in paths, so there's nothing higher than `DEL`.

### Write Permissions and Path Ownership

Paths can encode information about which authors are allowed to write to them.  Documents that break these rules are invalid and will be ignored.

A path is **shared** if it contains no `~` (tilde) characters.  Any author can write a document to a shared path.

A path is **owned** if it contains at least one `~`.  An author address immediately following a `~` is allowed to write to this path.  Multiple authors can be listed, each preceded by their own `~`, anywhere in the path.  The author address must begin with its usual leading `@`.

Example **shared** paths:

```
Anyone can write here:
/todos/123.json

Anyone can write here because there's no tilde "~"
/wall/@suzy.bo5sotcncvkr7p4c3lnexxpb4hjqi5tcxcov5b4irbnnz2teoifua/info.txt
```

Example **owned** paths:

```
Only suzy can write here:
/about/~@suzy.bo5sotcncvkr7p4c3lnexxpb4hjqi5tcxcov5b4irbnnz2teoifua/displayName.txt

Suzy and matt can write here, and nobody else can:
/chat/~@suzy.bo5sotcncvkr7p4c3lnexxpb4hjqi5tcxcov5b4irbnnz2teoifua~@matt.bwnhvniwd3agqclyxl4lirbf3qpfrzq7lnkzvfelg4afexcodz27a/messages.json
```

This path can't be written by anyone.  It's **owned** because it contains a tilde `~`, but an owner is not specified.  Even though the tilde appears without a `@` following it, it still acts as a marker of an owned path:

```
/nobody/can/ever/write/this/path/~
```

(TODO: loosen this restriction; treat `~` without a following `@` as a shared path?)

The `tilde + author address` pattern can occur anywhere in the path: beginning, middle or end.

Note that documents are mutable but their path can never change (or it would be a different document!) so the ownership of a particular path/document is permanent.  You can't change the ownership of a document; you have to create a new document at a different path.

### Path and Filename Conventions

Multiple apps can put data in the same workspace.  Here are guidelines to help them interoperate:

The first path segment SHOULD be a description of the data type OR the application that will read/write it.  Examples: `/wiki/`, `/chess/`, `/chat/`, `/posts/`, `/earthstagram/`, `/sillywiki/`.

> **Why?**
>
> Peers can selectively sync only certain documents.  Starting a path with a descriptive name like `/wiki/` makes it easy to sync only wiki documents and ignore the rest.  It also lets apps avoid accidentally reading or writing documents from other apps.

Sometimes this first path segment will represent a data type that many apps will support; sometimes it will be named after a specific app.

Consider including a version number in the path representing the version of the data format, like `/wiki-v1/` or `/wiki/v1/`.

Consider choosing a unique name for your app's data, like `/magic-todo-list/` instead of an obvious choice like `/todos/`, to avoid accidental collision with other apps you might not even know about.

### File Extensions

The last path segment SHOULD have a file extension to help applications know how to interpret the data.  For example, plain text documents should have paths ending in `.txt` and data encoded as JSON should have paths ending in `.json`.

Apps SHOULD use these file extensions to guess how to read and decode documents.

There is no way to explicitly signal that document content is binary (encoded as base64).  Applications will need to guess based on the file extension.  TODO: improve this.

See the [Content](#content) section for more details on base64 encoding of binary data.

## Documents and Their Fields

This example document is shown as JSON though it can exist in many serialization formats:

```json
{
  "author": "@suzy.bjzee56v2hd6mv5r5ar3xqg3x3oyugf7fejpxnvgquxcubov4rntq",
  "content": "Flowers are pretty",
  "contentHash": "bt3u7gxpvbrsztsm4ndq3ffwlrtnwgtrctlq4352onab2oys56vhq",
  "deleteAfter": null,
  "format": "es.4",
  "path": "/wiki/shared/Flowers",
  "signature": "bjljalsg2mulkut56anrteaejvrrtnjlrwfvswiqsi2psero22qqw7am34z3u3xcw7nx6mha42isfuzae5xda3armky5clrqrewrhgca",
  "timestamp": 1597026338596000,
  "workspace": "+gardening.friends",
}
```

Document schema in Typescript:

```ts
interface Doc {
    author: string, // an author address
    content: string, // an arbitary string of utf-8
    contentHash: string, // sha256(content) encoded as base32 with a leading 'b'
    deleteAfter: number | null,  // integer.  when the document expires.  null for non-expiring documents.
    format: 'es.4', // the format version that this document adheres to.
    path: string, // a path
    signature: string, // ed25519 signature of encoded document, signed by author
    timestamp: number, // integer.  when the document was created
    workspace: string, // a workspace address
}
```

Here we use the words "fields" and "properties" to mean the same thing.

The fields above are called the "core fields".  All core fields are REQUIRED.  Some core fields may be null; these MUST NOT be omitted; they MUST be exlicitly set to null if they are null.

Extra fields are FORBIDDEN as part of this core document schema, but are allowed during syncing -- see the **Extra Fields For Syncing** section below.

All string fields MUST BE limited to `PRINTABLE_ASCII` characters except for `content`, which is utf-8, or string fields can be null if specified above.  `PRINTABLE_ASCII` is defined earlier, and notably does not contain newline or tab characters, which are reserved for use in the serialization format we use for hashing and signing.

All number fields MUST BE integers, and cannot be NaN or Infinity, but they can be null if specified above.

The order of fields is unspecified except for hashing and signing purposes (see section below).  For consistency, the recommended canonical order is sorted lexicographically by field name.

### Extra Fields For Syncing

When sending documents over the network, peers may add additional properties as synchronization metadata. These extra fields MUST all have names beginning with an underscore to separate them from the core fields, and the the receiving peer MUST remove these fields before storing the document.

Likewise, when Earthstar libraries store documents internally, and send them over the network, they MAY add their own extra fields which MUST have names beginning with an underscore.

In other words, during storage the extra fields are "owned" by the local peer that's doing the storage.  During syncing, the fields are "owned" by the sending peer, and they are observed and removed by the receiving peer, which may in turn set its own extra fields before storing the document and/or before sending it again over the network.  The content of extra fields MUST NOT propagate or spread across the network of peers except for the brief moment of a single transmission, before they are removed by the receiving peer.

Extra fields are NOT included in the document's signature.

> **Example of extra fields for syncing:**
>
> These extra fields are not standardized yet.
>
> ```ts
> interface DocWithExtraFields extends Doc {
>     // (...The core fields will be present here also.)
>
>     // All extra fields must begin with an underscore.
>
>     // An integer >= 0, from the peer that sent the doc or
>     // or the peer storing it.  This represents the order
>     // in which the peer obtained the document compared to
>     // the other documents in its storage (in the same workspace).
>     // This is kept locally in storage, and sent when syncing.
>     // It's observed and removed from incoming docs when syncing
>     // (and replaced with the receiving peer's own localIndex value)
>     _localIndex: number,
>
>     // A unique ID of the storage instance that sent the doc.
>     // This is only present when the doc is being sent,
>     // it's not kept when the doc is at rest in storage.
>     _fromStorageId: string,
>
> }
> ```

### Document Validity

A document MUST be **valid** in order to be ingested into storage, either from a local write or a sync.

Invalid documents MUST be individually ignored when peers are syncing, and the sync MUST NOT be halted just because an invalid document was encountered.  Continue syncing in case there are more valid documents.

Documents can be temporarily invalid depending on their timestamp and the current wall clock.  Next time a sync occurs, maybe some of the invalid documents will have become valid by that time.

To be **valid** a document MUST pass ALL these rules, which are described in more detail in the following sections:

* `author` is a valid author address string
* `content` is a string holding utf-8 data
* `contentHash` is the sha256 hash of the `content`, encoded as base32 with a leading `b`, for a total length of 53 characters.
* `timestamp` is an integer between 10^13 and 2^53-2, inclusive
* `deleteAfter` is null, or is a timestamp integer in the same range as `timestamp`
* `format` is a string of printable ASCII characters
* `path` is a valid path string
* `signature` is a base32 string with a leading `b`.  For the `es.4` format it must be 104 characters long including the `b`.
* `workspace` is a valid workspace address string which matches the local workspace we are intending to write the document to
* No extra fields.  Any **extra fields for syncing** as described above should be removed before testing for validity.
* No missing fields
* Additional rules about `timestamp` and `deleteAfter` relative to the current wall clock (see below)
* Author has write permission to path based on tilde placement
* Signature is cryptographically valid

### Author

The `author` field holds an author address, formatted according to the rules described earlier in [Author Addresses](#author-addresses).

### Content

The `content` field contains arbitrary utf-8 encoded data.  If the data is not valid utf-8, the document is still considered valid but the library's behavior is undefined when trying to access the content.

The `content` field may be an empty string.  In fact, the recommended way to remove data from Earthstar is to overwrite the document with a new one which has `content = ""`.  Documents with empty-string content SHOULD be considered to be "deleted" and therefore omitted from some query results so that apps don't see them.  This depends on context; when syncing documents it's important to sync the "deleted" documents too because they act as tombstones.

The maximum length of the content field is 4 million bytes (4,000,000 bytes).  This is measured as "bytes when encoded as utf-8", not naive string length.  (This means the overall document, when encoded as JSON, can be slightly larger than 4 million bytes - the rest of the fields add about 450 bytes more.)

To store raw binary data in a utf-8 string, apps SHOULD encode it as base64.  In this case apps SHOULD put a file extension on the path that's well-known to be a binary format.  If you don't know what file extension to choose, use `.b64`.

When reading, apps SHOULD use the path's file extension to guess if a document contains textual data or base64-encoded binary data.

TODO: We will be adding support for binary data and a way to unambiguously know which data is binary and which is utf8.

> **Why no native support for binary data yet?**
>
> Common encodings such as JSON, and protocols built on them such as JSON-RPC and GraphQL, have to way to represent binary data, and we want to support the "least common denominator" standards that are widely used and known.


#### Sparse Mode (eventually)

We plan to add "sparse mode".  This will allow `content` to be `null`, meaning the local peer don't know what the content is.  This allows handling documents' metadata without their actual content, in case the peer wants to save space now and fetch the actual content later, perhaps on-demand.

This is not allowed yet; in the current version `content` MUST NOT be null.

Once [Sync Queries](#sync-queries) are implemented, peers will be able to sync the full content of small documents and get sparse versions of large documents.


It will be possible to verify the signature on sparse documents because it's based on the `contentHash`, which is always present.

### Content Hash

The `contentHash` is the `sha256` hash of the `content` data.  The hash digest is then encoded from binary to base32 following the usual Earthstar format, with a leading `b`.

Note that hash digests are usually encoded in hex format, but we use base32 instead to be consistent with the rest of Earthstar's encodings.

Wrong: `binary hash digest --> hex encoded string --> base32 encoded string`

Correct: `binary hash digest --> base32 encoded string`

Also be careful not to accidentally change the content string to a different encoding (such as utf-16) before hashing it -- hash the utf-8 bytes.

### Format

The format is a short string describing which version of the Earthstar specification to use when validating and interpreting the document.  It's like a schema name for the core Earthstar document format.

It MUST consist only of `PRINTABLE_ASCII` characters.

TODO: max length of format?

The current format version is `es.4` ("es" is short for Earthstar.)

If the specification is changed in a way that breaks forwards or backwards compatability, the format version MUST be incremented.  The version number SHOULD be a single integer, not a semver.

Other format families may someday exist, such as a hypothetical `ssb.1` which would embed Scuttlebutt messages in Earthstar documents, with special rules for validating the original embedded Scuttlebutt signatures as part of validating the document.

#### Format Validator Responsibilities

Earthstar libraries SHOULD separate out the code related to each format version, so that they can handle old and new documents side-by-side.  Code for handling a format version is called a **Validator**.  Validators are responsible for:

* Hashing documents
* Signing new documents
* Checking document validity when ingesting documents.  See the [Document Validity](#document-validity) section for more info.

Therefore each different format can have different ways of hashing, signing, and validating documents.

TODO: define basic rules that documents of all formats must follow

### Path

The `path` field is a string following the rules described in [Paths](#paths).

The document is invalid if the author does not have permission to write to the path, following the rules described in [Write Permissions and Path Ownership](#write-permissions-and-path-ownership).

The path MUST contain at least one `!` character, anywhere, IF AND ONLY IF the document is ephemeral (has non-null `deleteAfter`).

### Timestamp

Timestamps are integer **microseconds** (millionths of a second) since the Unix epoch.

Note this is NOT the default format used by Javascript, which uses milliseconds (thousandths of a second).

```ts
// Earthstar timestamps in javascript
let timestamp = Date.now() * 1000;
```

```python
# Earthstar timestamps in python
timestamp = int(time.time() * 1000 * 1000)
```

Timestamps MUST be within the following range (inclusive):

```ts
// 10^13
let MIN_TIMESTAMP = 10000000000000

// 2^53 - 2  (Javascript's Number.MAX_SAFE_INTEGER - 1)
let MAX_TIMESTAMP = 9007199254740990

let timestampIsValid =
    MIN_TIMESTAMP <= timestamp
    && timestamp <= MAX_TIMESTAMP;
```

> **Why this specific range?**
>
> The min timestamp is chosen to reject timestamps that were accidentally computed in milliseconds or seconds.
>
> The max timestamp is the largest safe integer that Javascript can represent.
>
> The range of valid times is approximately 1970-04-26 to 2255-06-05.

Timestamps MUST NOT be from the future from the perspective of the peer accepting them in a sync; but a limited tolerance is allowed to account for clock skew between devices.  The recommended value for the future tolerance threshold is 10 minutes, but this can be adjusted depending on the clock accuracy of devices in a deployment scenario.

Timestamps from the future, beyond the tolerance threshold, are (temporarily) invalid and MUST NOT be accepted in a sync.  They can be accepted later, after they are no longer from the future.

> **Choosing the future tolerance threshold**
>
> In some settings such as in-the-field embedded devices, where devices do not have accurate clocks or connectivity to NTP servers, the future tolerance may be greatly increased.  However this enables some possible attacks on the network that can cause instability, so it requires greater trust in the network participants.  In extreme cases we may need to add algorithms for the peers to attempt to converge on a rough understanding of the current time to account for clock skew.
>
> In these scenarios, document timestamps should be considered more like version numbers rather than actual meaningful timestamps.

Also see the (non-normative) document [How does Earthstar handle timestamps, and can it recover from a device with a very inaccurate clock?](https://github.com/earthstar-project/earthstar/blob/main/docs/timestamps.md)

### Ephemeral documents: deleteAfter

Documents may be regular or ephemeral.  Ephemeral documents have an expiration date, after which they MUST be proactively deleted by Earthstar libraries.

> **Why have ephemeral documents?**
>
> Deleting a regular document leaves behind a small empty document which takes up space.  Ephemeral documents are completely removed when they expire, so they are a good choice for applications which will write many short-lived documents.
>
> They also provide more privacy.  Users can always delete their regular documents, but that deletion must propagate across all the peers.  Ephemeral documents will be deleted from the entire network when they expire even if some peers have lost connectivity or you are not there to request a deletion at that time.

Libraries MUST check for and delete all expired documents at least once an hour (while they are running).  Deleted documents MUST be actually deleted, not just marked as ignored.

Libraries MUST filter out expired documents from queries and lookups and not return them.  Libraries MAY or MAY NOT actually delete them when they are encountered during querying; they may choose to wait until the next scheduled hourly deletion time.

Expired documents MUST not be sent or accepted during a sync.  Both peers in a sync should filter the incoming and outgoing documents to enforce this.  This is the responsibility of the [Format Validator](#format).

The `deleteAfter` field holds the timestamp after which a document is to be deleted.  It is a timestamp with the same format and range as the regular `timestamp` field.

Regular, non-ephemeral documents have `deleteAfter: null`.  The field is still required and may not be omitted.

Unlike the `timestamp` field, the `deleteAfter` field is expected to be in the future compared to the current wall-clock time.  Once the `deleteAfter` time is in the past, the document becomes invalid.

The `deleteAfter` time MUST BE strictly greater than the document's `timestamp`.

The document path MUST contain at least one exclamation mark `!` character IF AND ONLY IF the document is ephemeral.  Regular, non-ephemeral documents MUST NOT have any `!` characters in their paths.

Ephemeral documents MAY be edited by users to change the expiration date.  This works best if the expiration date is increased into the future.  If it's decreased so it expires sooner, the document may sync in unpredictable ways (see below for another example of this).  If it's set to expire in the past, the document won't even sync off of the current peer because other peers will reject it, so the edit won't propagate.  When shortening the expiration date there should be time for the edit to propagate across the entire network of peers before the document expires.

> **Why ephemeral documents need a `!` in their path**
>
> Regular and ephemeral documents with the same path could interact in surprising ways.  To avoid this, we enforce that they can never collide on the same path.
>
> (An ephemeral document could propagate halfway across a network of peers, overwriting a regular document with the same path, and then expire and get deleted wherever it has spread.  Then the regular document would regrow to fill the empty space.
>
> But if the ephemeral document traveled across the entire network and exterminated the regular document, and THEN expired, there would be nothing left.
>
> Which of these cases occurred would depend on how long the document took to spread, which could be very fast or could take months if there was a peer that was usually offline.  We'd like to avoid this unpredictability.)

### Signature

The ed25519 signature by the author, encoded in base32 with a leading `b`.

See [Serialization for Hashing and Signing](#serialization-for-hashing-and-signing), below, for details.

Like the hashes and crypto keys in Earthstar, this is the raw binary signature encoded directly into base32.  Do not encode the binary signature into a hex string and then into base32.

### Workspace

The `workspace` field holds a workspace address, formatted according to the rules described in [Workspace Addresses](#workspace-addresses).

Each document belongs to exactly one workspace and cannot be moved to another workspace (because that would cause the signature to become invalid).

## Document Serialization

There are 3 scenarios when we need to serialize a document to/from a series of bytes:

* Hashing and signing
* Network transmission
* Storage

They have different needs and we use different formats for each.

### Serialization for Hashing and Signing

When an author signs a document, they're actually signing a hash of the document.  We need a deterministic, standardized, and simple way to serialize a document to a sequence of bytes that we can hash.  This is a **one-way** conversion -- we never need to deserialize this format.

Earthstar libraries MUST use this exact process.

To hash a document:

```ts
// Pseudocode

let hashDocument(document): string => {
    // Get a deterministic hash of a document as a base32 string.
    // Preconditions:
    //   All string fields must be printable ASCII only
    //   Fields must have one of the following types:
    //       string
    //       string | null
    //       integer
    //       integer | null
    //   Note that "string | integer" is not allowed
    //   because we'd have no way of telling "123" apart from 123.

    let accum: string = '';

    For each field and value in the document, sorted in lexicographic order by field name: {

        // Skip the content and signatue fields
        if (field === 'content' || field === 'signature') { continue; }

        // Skip null fields
        if (value === null) { continue; }

        // Otherwise, append the fieldname and value.
        // Tab and newline are our field separators.
        // Convert integers to strings here.
        accum += fieldname + "\t" + value + "\n"

        // (The newline is included on the last field.)
    }

    // Binary digest, not hex digest string!
    let binaryHashDigest = sha256(accum).digest();

    // Convert bytes to Earthstar b32 format with leading 'b'
    return base32encode(binaryHashDigest);
}
```

To sign a document:

```ts
// Pseudocode

let signDocument(authorKeypair, document): void => {
    // Sign the document and store the signature into the document (mutating it).
    // authorKeypair contains a pubkey and a private key.

    let binarySignature = ed25519sign(
        authorKeypair,
        hashDocument(document)
    );

    // Convert bytes to Earthstar b32 format with leading 'b'
    let base32signature = base32encode(binarySignature);

    document.signature = base32signature;
}
```

Preconditions that make this work:
* Documents can only hold integers, strings, and null -- no floats or nested objects that could increase complexity or be nondeterministic
* No document field name or field content can contain `\t` or `\n`, except `content`, which is not directly used (we use `contentHash` instead).  So we can safely use tab and newline as field separators.
* We don't need to worry about telling strings and integers apart because each field can hold an integer, or a string, but not both.  So we don't need to quote our strings with quote marks.

The reference implementation is in `hashDocument()` in `src/validators/es4.ts`.  Here's a summary:

```ts
// Typescript-style psuedocode

let serializeDocumentForHashing = (doc: Document): string => {
    // We've hardcoded the algorithm here since we
    //  know what the fields are.
    // Fields are in lexicographic order.
    // Convert numbers to strings.
    // Omit properties that are null.
    // Use the contentHash instead of the content.
    // Omit the signature.
    return (
        `author\t${doc.author}\n` +
        `contentHash\t${doc.contentHash}\n` +
        (doc.deleteAfter === null
            ? ''
            : `deleteAfter\t${doc.deleteAfter}\n`) +
        `format\t${doc.format}\n` +
        `path\t${doc.path}\n` +
        `timestamp\t${doc.timestamp}\n` +
        `workspace\t${doc.workspace}\n`
        // Note the \n is included on the last item too.
    );
}

let hashDocument = (doc: Document): Base32String => {
    return bufferToBase32(
        sha256AsBuffer(
            serializeDocumentForHashing(doc)
        )
    );
}

let signDocument = (keypair: AuthorKeypair, doc: Document): Document => {
    return {
        ...doc,
        signature: sign(keypair, hashDocument(doc))
    };
}
```

Actual example data:

```
INPUT (shown as JSON, but is actually in memory before serialization)

{
  "format": "es.4",
  "workspace": "+gardening.friends",
  "path": "/wiki/shared/Flowers",
  "contentHash": "bt3u7gxpvbrsztsm4ndq3ffwlrtnwgtrctlq4352onab2oys56vhq",
  "content": "Flowers are pretty",
  "author": "@suzy.bjzee56v2hd6mv5r5ar3xqg3x3oyugf7fejpxnvgquxcubov4rntq",
  "timestamp": 1597026338596000,
  "deleteAfter": null,
  "signature": ""  // is empty before signing has occurred
}

SERIALIZED FOR HASHING
(tabs and newlines should be real but
 are represented here as \t \n):

author\t@suzy.bjzee56v2hd6mv5r5ar3xqg3x3oyugf7fejpxnvgquxcubov4rntq\n
contentHash\tbt3u7gxpvbrsztsm4ndq3ffwlrtnwgtrctlq4352onab2oys56vhq\n
format\tes.4\n
path\t/wiki/shared/Flowers\n
timestamp\t1597026338596000\n
workspace\t+gardening.friends\n

HASHED WITH SHA256 AND ENCODED AS BASE32:

b6nyw25gum45gcxbhez3ykx3jopkhlfjj2rnmfb7rt6yhkszvidsa

AUTHOR KEYPAIR:
(author address must match the author in the document)

{
  "address": "@suzy.bjzee56v2hd6mv5r5ar3xqg3x3oyugf7fejpxnvgquxcubov4rntq",
  "secret": "b6jd7p43h7kk77zjhbrgoknsrzpwewqya35yh4t3hvbmqbatkbh2a"
}

SIGNATURE AS BASE32:

bjljalsg2mulkut56anrteaejvrrtnjlrwfvswiqsi2psero22qqw7am34z3u3xcw7nx6mha42isfuzae5xda3armky5clrqrewrhgca

FINAL SIGNED DOCUMENT:

{
    ...same as the original, except with signature:

    "signature": "bjljalsg2mulkut56anrteaejvrrtnjlrwfvswiqsi2psero22qqw7am34z3u3xcw7nx6mha42isfuzae5xda3armky5clrqrewrhgca"
}
```

> **Why use `contentHash` instead of `content` for hashing documents?**
>
> This lets us drop the actual content (to save space) but still verify the document signature.  This will be useful in the future for "sparse mode".

### Serialization for Network

This is a **two-way** conversion between memory and bytes.

Earthstar doesn't have strong opinions about networking.  This format does not need to be standardized, but it's good to choose widely used familiar tools.  

Apps and libraries SHOULD use JSON (encoded as UTF-8) as a default choice unless there are important reasons to choose otherwise.  JSON is widely known, widely supported, and fits within most network protocols easily.

**Other options to consider**:

* Encodings
  * JSON
  * newline-delimited JSON for streaming lots of documents
  * CBOR
  * msgpack
* Protocols
  * REST
  * JSON-RPC
  * GraphQL (relies on JSON)
  * gRPC?
  * muxrpc (from SSB)

### Serialization for Storage

This is a **two-way** conversion between memory and bytes.

It does not need to be standardized; each implementation can use its own format.

It needs to support efficient mutation and deletion of documents, and querying by various properties.

It would be nice if this was an archival format (corruption-resistant and widely known).

**Options to consider:**

* SQLite
* Postgres
* IndexedDB
* leveldb or similar key-value databases (with extra indexes)
* a bunch of JSON files, one for each document (with extra indexes)

For exporting and importing data:
* one giant newline-delimited JSON file, one document per line, is easier to parse than a giant JSON array of documents.

## Querying

Libraries SHOULD support a standard variety of queries against a database of Earthstar messages.  A query is specified by a single object with optional fields for each kind of query operation.

This query format will become standardized because it will be used for querying from one peer to another.  It's not quite stable yet.

This only supports relatively simple ways of querying and filtering documents because we want to make it easy to use many different kinds of backend storage which may have limited query capabilities.  Apps and libraries MAY add extensions for more powerful querying if they're able to, but this should be considered the minimal set for compatability across peers.

The recommended query object format, expressed in Typescript, is... (see [query.ts](https://github.com/earthstar-project/earthstar/blob/main/src/storage/query.ts) for the latest version of this):

```ts
/**
 * Query objects describe how to query a Storage instance for documents.
 * 
 * An empty query object returns all latest documents.
 * Each of the following properties adds an additional filter,
 * narrowing down the results further.
 * The exception is that history = 'latest' by default;
 * set it to 'all' to include old history documents also.
 * 
 * HISTORY MODES
 * - `latest`: get latest docs, THEN filter those.
 * - `all`: get all docs, THEN filter those.
 */
export interface Query {
    //=== filters that affect all documents equally within the same path

    /** Path exactly equals... */
    path?: string,

    /** Path begins with... */
    pathStartsWith?: string,

    /** Path ends with this string.
     * Note that pathStartsWith and pathEndsWith can overlap: for example
     * { pathStartsWith: "/abc", pathEndsWith: "bcd" } will match "/abcd"
     * as well as "/abc/xxxxxxx/bcd".
     */
    pathEndsWith?: string,

    //=== filters that differently affect documents within the same path

    /** Timestamp exactly equals... */
    timestamp?: number,

    /** Timestamp is greater than... */
    timestampGt?: number,

    /** Timestamp is less than than... */
    timestampLt?: number,

    /**
     * Document author.
     * 
     * With history:'latest' this only returns documents for which
     * this author is the latest author.
     * 
     * With history:'all' this returns all documents by this author,
     * even if those documents are not the latest ones anymore.
     */
    author?: AuthorAddress,

    contentLength?: number,  // in bytes as utf-8.  TODO: how to treat sparse docs with null content?
    contentLengthGt?: number,
    contentLengthLt?: number,

    //=== other settings

    /**
     * If history === 'latest', return the most recent doc at each path,
     * then apply other filters to that set.
     * 
     * If history === 'all', return every doc at each path (with each
     * other author's latest version), then apply other filters
     * to that set.
     * 
     * Default: latest
     */
    history?: HistoryMode,

    /**
     * Only return the first N documents.
     * There's no offset; use continueAfter instead.
     */
    limit?: number,

    /**
     * Accumulate documents until the sum of their content length <= limitByts.
     * 
     * Content length is measured in UTF-8 bytes, not unicode characters.
     * 
     * If some docs have length zero, stop as soon as possible (don't include them).
     */
    limitBytes?: number,

    /**
     * Begin with the next matching document after this one:
     */
    continueAfter?: {
        path: string,
        author: string,
    },
};
```

## Syncing

Syncing is the process of trading documents between two peers to bring each other up to date.

Syncing can occur locally (within a process, between two Storage instances) as well as across a network.

Documents are locked into specific workspaces; therefore syncing can't transfer documents between workspaces, only between different peers that hold the same workspace.

### Networking

The network protocols used by peers to sync documents are not standardized yet.

### Workspace Secrecy

Knowing a workspace address gives a user the power to read and write to that workspace, so they need to be kept secret.

It MUST be impossible to discover new workspaces through the syncing process.  Peers MUST keep their workspaces secret and only transmit data when they are sure the other peer also knows the address of the same workspace.

Here's an algorithm to exchange common workspaces without discovering new ones:

* Peer1 and Peer2 each generate a random string of entropy and send the entropy to each other.
* Each peer shares `sha256(workspaceAddress + entropyFrom1 + entropyFrom2)` for each of their workspaces

The hashes they have in common correspond to the workspaces they both know about.

The hashes that are unique to one peer will reveal no information to the other peer, except how many workspaces the other peer has.

They can now proceed to sync each of their common workspaces, one at a time.

#### Eavesdropping

An eavesdropper observing this exchange will know both pieces of entropy, and can confirm that the peers have or don't have workspaces that the eavesdropper already knows about, but can't un-hash the exchanged values to get the workspace addresses they don't already know.

But once the peers start trading actual workspace data, an eavesdropper can observe the workspace addresses in plaintext in the exchanged documents.

Peers SHOULD talk to each other over an encrypted connection such as HTTPS or [secret-handshake](https://www.npmjs.com/package/secret-handshake).

### Sync Queries

(This feature is not implemented yet.)

During a sync, apps SHOULD be able to specify which documents they're willing to share, and which they're interested in getting.

Apps do this by defining **Sync queries**.  An app SHOULD be able to define:

* An array of **incoming sync queries** -- what you want
* An array of **outgoing sync queries** -- what you will share

The queries in each array are additive: if a doument matches any query in the array, the document is chosen.

This contrasts with the behavior of query fields inside each query object, each of which narrows the results down further.

By taking advantage of these two techniques, both AND and OR type behaviour is possible.

Example:
```ts
// ask for all the About documents, and the recent Wiki documents
let incomingQueries = [
    {
        pathStartsWith: '/about/',
        history: 'all',
    },
    {
        pathStartsWith: '/wiki/',
        history: 'all',
        tiemstampGt: 15028732938984
    },
];

// only share documents I wrote myself
let outgoingQueries = [
    {
        author: '@suzy.bjzee56v2hd6mv5r5ar3xqg3x3oyugf7fejpxnvgquxcubov4rntq',
        history: 'all',
    },
];
```

(Sync queries are is not implemented yet as of March 2021.)

### Resolving Conflicts

See the [Data model](#data-model) section for details about conflict resolution.

## Future Directions

These are not implemented or specified yet:

### Sparse Mode

Allow handling documents without their content.  Then the content can be fetched later, on-demand.  This is especially useful for large documents that you don't always want, like image files in a wiki.

Sparse documents would have `content: null` meaning the content is unknown or absent.  It's different than `content: ''` which means the document is empty.

Doc signatures can be verified without having the content, because we only include the contentHash in the signature.

See more [in this issue.](https://github.com/earthstar-project/earthstar/issues/33)

### Invite-Only Workspaces

Right now anyone can write to a workspace if they know its address.

We can add a second kind of workspace which has a pubkey in its address. Every item posted in this kind of workspace must be signed by the workspace private key AND the author private key. This will be enforced by the feed format validator class.

So to write you need to know the workspace private key. You can give the workspace secret to someone to invite them.

Invite-only workspaces could also have their documents encrypted to the public key of the workspace.  Then only people who know the workspace key can read the documents.

This separates permissions into two tiers depending on whether or not you know the workspace private key:
1. sync all documents (read-only), and only understand the unencrypted ones
2. also understand all documents, and be able to write documents.

And that opes up new possibilities:
* Peers and pubs can host workspaces without being able to edit them or maybe even understand them
* A few authors could publish a workspace that many people could read but not edit.  It could be a blog, or it could contain the code files for an Earthstar app.

See more [in this issue.](https://github.com/earthstar-project/earthstar/issues/7)

### Transport Encryption

Peers could encrypt their communications with SSL, Noise Protocol, etc.

### Document Encryption

The document `content` field can be encrypted by apps in any way they like.  We have some convenient keys available already:
* You can write a private message to one author using their public key
* You can write a private message to the entire workspace using the workspace public key (if it's an invite-only workspace)

We could use a format like [private-box](https://www.npmjs.com/package/private-box) to allow multiple recipients.

However, the rest of the document metadata will be in plaintext including the author, path, and timestamp, which might reveal important information.  App authors would have to use uninformative paths.  [This issue](https://github.com/earthstar-project/earthstar/issues/11) discusses ways of nesting the metadata inside another document to obscure it.

### Immutable Documents

Documents that can't be edited.  They may or may not be able to be deleted, and they may or may not be ephemeral (expiring).

This would probably involve a new optional document field, `immutable`.

See more [in this issue.](https://github.com/earthstar-project/earthstar/issues/8).
