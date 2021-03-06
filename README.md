# kappa-core

> a small core for append-only log based programs

A lot like [flumedb][flumedb], but using
[multifeed](https://github.com/noffle/multifeed) as an append-only log base,
which is actually a *set* of append-only logs.

Pronounced *"capricorn"*.

## Status

*Experimental*, but functional.

## Usage

```js
var kappa = require('kappa-core')
var memdb = require('memdb')

var core = kappa('./log', { valueEncoding: 'json' })
var idx = memdb()

var sum = 0

var sumview = {
  api: {
    get: function (core, cb) {
      this.ready(function () {
        cb(null, sum)
      })
    }
  },
  map: function (msgs, next) {
    msgs.forEach(function (msg) {
      if (typeof msg.value === 'number') sum += msg.value
    })
    next()
  },

  // where to store and fetch the indexer's state (which log entries have been
  // processed so far)
  storeState: function (state, cb) { idx.put('state', state, cb) },
  fetchState: function (cb) { idx.get('state', cb) }
}

// the api will be mounted at core.api.sum
core.use('sum', sumview)

core.feed('default', function (err, feed) {
  feed.append(1, function (err) {
    core.api.sum.get(function (err, value) {
      console.log(value) // 1
    })
  })
})
```

## API

```js
var kappa = require('kappa-core')
```

### var core = kappa(storage, opts)

Create a new kappa-core database.

- `storage` is an instance of
  [random-access-storage](https://github.com/random-access-storage). If a string
  is given,
  [random-access-file](https://github.com/random-access-storage/random-access-storage)
  is used with the string as the filename.
- Valid `opts` include:
  - `valueEncoding`: a string describing how the data will be encoded.

### var feed = core.feed(name, cb)

Create or get a local writable feed called `name`. If it already existed, it is
returned. A feed is an instance of
[hypercore](https://github.com/mafintosh/hypercore).

### core.use(name, view)

Install a view called `name` to the kappa-core instance. A view is an object of
the form

```js
{
  api: {
    someSyncFunction: function (core) { return ... },
    someAsyncFunction: function (core, cb) { process.nextTick(cb, ...) }
  },

  map: function (msgs, next) {
    msgs.forEach(function (msg) {
      // ...
    })
    next()
  },

  fetchState: function (cb) { ... },
  storeState: function (state, cb) { ... }
}
```

The kappa-core instance `core` is always is bound to `this` in all of the `api`
functions you define.

The `{fetch,store}State` functions are optional: they tell the view where to
store its state information about what log entries have been indexed thus far.
If not passed in, they will be stored in memory (i.e. reprocessed on each fresh
run of the program). You can use any backend you want (like leveldb) to store
the `Buffer` object `state`.

### core.ready(viewNames, cb)

Wait until all views named by `viewNames` are caught up. e.g.

```
// one
core.ready('sum', function () { ... })

// or several
core.ready(['kv', 'refs', 'spatial'], function () { ... })
```

If viewNames is `[]` or not included, all views will be waited on.

### core.replicate([opts])

Create a duplex replication stream. `opts` are passed in to
[multifeed](https://github.com/noffle/multifeed)'s API of the same name.

## Install

With [npm](https://npmjs.org/) installed, run

```
$ npm install kappa-core
```

## Why?

[flumedb][flumedb] presents an ideal small core API for an append-only log:
append new data, and build (versioned) views over it. kappa-core copies this
gleefully, but with two major differences:

1. [hypercore][hypercore] is used for feed (append-only log) storage
2. views are built in out-of-order sequence

hypercore provides some very useful superpowers:

1. all data is cryptographically associated with a writer's public key
2. partial replication: parts of feeds can be selectively sync'd between peers,
instead of all-or-nothing, without loss of cryptographic integrity

Building views in arbitrary sequence is more challenging than when order is
known to be topographic, but confers some benefits:

1. most programs are only interested in the latest values of data; the long tail
of history can be traversed asynchronously at leisure after the tips of the
feeds are processed
2. the views are tolerant of partially available data. Many of the modules
listed in the section below depend on *topographic completeness*: all entries
referenced by an entry **must** be present for indexes to function. This makes
things like the equivalent to a *shallow clone* (think [git][git-shallow]),
where a small subset of the full dataset can be used and built on without
breaking anything.

## Acknowledgments

kappa-core is built atop ideas from a huge body of others' brilliant work:

- [flumedb][flumedb]
- [secure scuttlebutt](http://scuttlebutt.nz)
- [hypercore](https://github.com/mafintosh/hypercore)
- [hyperdb](https://github.com/mafintosh/hyperdb)
- [forkdb](https://github.com/substack/forkdb)
- [hyperlog](https://github.com/mafintosh/hyperlog)

## Further Reading

- [kappa architecture](http://kappa-architecture.com)

## License

ISC

[flumedb]: https://github.com/flumedb/flumedb
[git-shallow]: https://www.git-scm.com/docs/gitconsole.log(one#gitconsole.log(one---depthltdepthgt)
