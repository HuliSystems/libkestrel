
# libkestrel

Libkestrel is a library for scala/java containing:

- ConcurrentBlockingQueue - a lock-free queue that allows readers to block
  (wait) for items

- JournaledQueue - a queue with potentially many read-pointers, where state
  of the queue and its readers is saved to disk

These are variants and improvements of the building blocks of the kestrel
distributed queue server project. The intent (as of November 2011) is to make
kestrel use this library in its next major release.


## Build

    $ sbt clean update package-dist

SBT must presently be version 0.11.2 or you'll get build errors (due to SBT
plugin versioning). Note also that if the script you use to run SBT ("sbt"
in the example) enables assertions (via java -ea or java -enableassertions)
the build will fail due to a Scala compiler bug.

## Community

Come talk to us on the kestrel mailing list!

http://groups.google.com/group/kestrel-talk


## ConcurrentBlockingQueue

ConcurrentBlockingQueue extends the idea of java's `ConcurrentLinkedQueue` to
allow consumers to block, indefinitely or with a timeout, until items arrive.

It works by having one `ConcurrentLinkedQueue` for the queue itself, and
another one to track waiting consumers. Each time an item is put into the
queue, it's handed off to the next waiting consumer, like an airport taxi
line.

The handoff occurs in a serialized block (like the `Serialized` trait in
util-core), so when there's no contention, a new item is handed directly from
the producer to the consumer. When there is contention, producers increment a
counter representing how many pent-up handoffs there are, and the producer
that got into the serialized block first will do each handoff in order until
the count is zero again. This way, no producer is ever blocked.

Consumers receive a future that will eventually be fulfilled either with
`Some(item)` if an item arrived before the requested timeout, or `None` if the
request timed out. If an item was available immediately, the future will be
fulfilled before the consumer receives it. This way, no consumer is ever
blocked.


## JournaledQueue

A JournaledQueue is a journaled queue that may have multiple "readers", each of
which may have multiple consumers.

### Puts

When an item is added to a queue, it's journaled and notifiation is passed on
to any readers.  There is always at least one reader, and the reader knows its
current location in the memory-mapped journal file. If there are multiple
readers, they behave as multiple independent queues, each receiving a copy of
each item added to the `JournaledQueue`, but sharing a single journal. They may
have different policies on queue size limits, item expiration, and error
handling.

### Gets

Items are read only from readers. When an item is available, it's set aside as
an "open read", but not committed to the journal. A separate call is made to
either commit the item or abort it. Aborting an item returns it to the head of
the queue to be given to the next consumer.

Periodically each reader records its state in a separate checkpoint journal.
When initialized, if a journal already exists for a queue and its readers, each
reader restores itself from this saved state. If the queues were not shutdown
cleanly, the state files may be out of date and items may be replayed. Care is
taken never to let any of the journal files be corrupted or in a
non-recoverable state. In case of error, the choice is always made to possibly
replay items instead of losing them.

### Infinite scroll

The writer journal is treated as an "infinite scroll" of put operations. Each
journal file is created with the current timestamp in its filename, and after
it gets "full" (configurable, but 16MB by default), that file is closed and a
new one is opened. If no readers ever consumed items from the queue, these
files would sit around forever.

Once all readers have moved their read-pointer (the head of their queue) past
the end of journal file, that file is archived. By default, that just means
the file is deleted, but one of the configuration parameters allows you to
have the dead files moved to a different folder instead.

There are several advantages to splitting the journals into a single
(multi-file) writer journal and several reader checkpoint files:

- Fan-out queues (multiple read-pointers into the same queue) are free, and
  all the readers share a single journal, saving disk space and bandwidth.
  Disk bandwidth is now almost entirely based on write throughput, not the
  number of readers.

- The journals never have to be "packed" to save disk space, the way they did
  in kestrel 2.x. Packing creates more disk I/O at the very time a server
  might be struggling to keep up with existing load.

- Archiving old queue items is trivial, and allows you to do some
  meta-analysis of load offline.


## Tests

Some load tests are included. You can run them with

    $ ./dist/libkestrel/scripts/qtest

which will list the available tests. Each test responds to "`--help`".


## File by file overview

- BlockingQueue - interface for any blocking queue

- SimpleBlockingQueue - a simple queue using "synchronized", based on the one
  in kestrel 2.1

- ConcurrentBlockingQueue - a lock-free BlockingQueue (see above)

- PeriodicSyncFile - a writable file that `fsync`s on a schedule

- ItemIdList - a simple Set[Long] implementation optimized for small sets

- QueueItem - an item plus associated metadata (item ID, expiration time)

- JournalFile - a single on-disk journal file, encapsulating reading & writing
  journal records

- Journal - representation of a collection of files (the writer files and a
  file for each reader)

- JournaledQueue - a `Journal` based queue implementation

- JournaledBlockingQueue - JournaledQueue wrappers that provide a simplified
  interface for users that only use a single reader (with or without
  transactions)

## Improvement targets

- Trustin suggested that the read-pointer files could be, instead of an id, a
  filename and position. That would save us from reading the first half of a
  journal file on startup (to find the id).

- Nick suggested that writing all of the readers into the same file could
  reduce disk I/O by writing fewer blocks during reader checkpointing.
