# Consistency & Atomicity

One of the challenges of databases is consistency and atomicity. What is that?

If you make a change to a record and then your program crashes, you expect that it's either completely written to the database or not at all – having the first bytes of your change overwriting the previous state of the record is a no-go.

## The in-memory approach

In-memory databases can take an easy approach to solve that problem – they can just add all changes to the end of the file. Have a look at a sembast database file where record A got added, record B got added and then record A got removed:

```json
{"key":"a","store":"box","value":{"foo":1}}
{"key":"b","store":"box","value":{"foo":1}}
{"key":"a","store":"box","value":"deleted"}
```

The record still exists in the file, it's just not shown to the application any more.
Of course, if you change records a lot and remove records, the file gets very large.
To combat this, sembast (and also other in-memory databases, like Hive) introduce a "compaction" phase: They simply read all records into memory, create a new file (something like `database.compacted.json`), write the newest version of all existing records into that file, remove the old database and then remove the new version so that it becomes the de-facto database.

That approach is simple – just throw everything away and build it new – but it's effective.
It does have some disadvantages though:

* You need to choose the right time to do compaction. For example, something like "if 50 % of all records are either overwritten or deleted, compact everything". Every dataset differs, so while these heuristics fit most cases, there can always be some where it doesn't achieve optimal results.
* It doesn't scale. For large datasets, holding all values in memory isn't possible.
* Compaction takes time. This makes the database unpredictable – similar to how in most garbage-collected languages, the program is completely stopped for a moment to look for unused objects, compaction can also cause unpredictable spikes in resource exhausting. That's not as bad as garbage collection, because it doesn't stop everything (I/O is asynchronous, but it still takes more time to guarantee that a value has been written to disk).

## Chest's approach

For big amounts of data, it's not viable to simply throw out everything and rebuild the database from scratch.
To be able to do incremental updates, chunks are introduced. They are contiguous slices of memory of a certain size.
Chest's lowest layer – chunky – provides an abstraction from these chunks so you can atomically write to multiple chunks.

```dart
final chunky = await Chunky.named('sample');
final firstChunk = await chunky.read(0);

// Writes on chunks only change the data in memory. To actually write them to
// disk, `chunky.write` has to be used inside a transaction.
firstChunk.setUint8(24, 42);

await chunky.transaction((chunky) async {
  await chunky.write(0, firstChunk);
});

final newChunk = Chunk.empty();
newChunk.setUint8(1, firstChunk.getUint8(2));

// Inside transactions, multiple chunks can be written to.
await chunky.transaction((chunky) async {
  final chunkId = await chunky.add(newChunk);
  firstChunk.setUint8(123, chunkId);
  await chunky.write(0, firstChunk);
});
```

As you see, all writes happen inside transactions and chunks cannot be modified, only completely rewritten to disk.
Before the changes are actually applied to the `sample.chest` file, they are written to a second `sample.chest.transaction` file.
So, a chunk is actually written two times. The benefit is that writing becomes atomic: If the program crashes while writing a change, the change has already been writing to the transaction file, so it can be restored.