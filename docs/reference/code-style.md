---
title: Coding Style
---

## Guidelines for contributing to Earthstar

**Read [CONTRIBUTING.md](https://github.com/earthstar-project/earthstar/blob/main/CONTRIBUTING.md)** which has more details than this shortened guide.

CONTRIBUTING.md also discusses Purpose, Values, Code of Conduct, etc.

## Code style for the core Earthstar library

Note that we have Prettier set up but we don't really use it; we should remove it.

### Indentation and line length.

Indent with 4 spaces, just because.

Long lines are ok; don't contort yourself to achieve short lines.

It's ok to pack short if-statements into a single line, like:

```ts
    if (x === 0) { return; }
```

### Prefer `null` to `undefined`

`undefined` can't be JSON-serialized, and it's more likely to occur accidentally (e.g. because of a failed lookup).

In rare cases where you're mimicking an existing JS API such as Map, you can use `undefined` to indicate something was not found.

### Prefer to return exceptions instead of throwing them.

This makes Typescript aware of the exceptions that might come out of a function, forcing you to deal with them.

Try to use the custom Earthstar error classes [defined in `/src/util/types.ts`](https://github.com/earthstar-project/earthstar/blob/main/src/util/types.ts).

You can use the utility functions `isErr()` and `notErr()` to test if something is an instance or subclass of `EarthstarError`.  They're defined in the same file as the errors, above.

### Prefer multi-line imports

If there are 2 or more items imported from a file, put them each on their own line in alphabetical order.

If there's only one item, it can follow the above pattern or it can be condensed into a single line.

```ts
import {
    AuthorAddress,  // in alphabetical order
    AuthorKeypair,
    Path,
    WorkspaceAddress,
} from '../util/types';
import { sign } from '../crypto/crypto';
```
