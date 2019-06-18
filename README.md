LMDB is probably the fastest and most efficient database on the planet, if used correctly. `lmdb-store` provides a simple interface for interacting with LMDB, as a key-value store, that makes it easy to properly leverage the power, crash-proof design, and efficiency of LMDB. Used directly, LMDB has certain characteristics that can be very challenging, including fixed db size, inefficieny with flushing small transactions, and complicated cursors usage. `lmdb-store` offers several key features that make it NodeJS idiomatic, highly performant, and easy to use than LMDB efficiently:
* Automated database size handling
* Queueing asynchronous write operations with promise-based API
* Transaction management
* Iterable queries/cursors

`lmdb-store` is build on the excellent `node-lmdb` package.

## Design
<a href="https://dev.doctorevidence.com/"><img src="./assets/powers-dre.png" width="203" /></a>
When an `lmdb-store` is created, an LMDB environment/database is created, and starts with a default DB size of 1MB. LMDB itself uses a fixed size, but `lmdb-store` detects whenever the database goes beyond the current size, and automatically increases the size of DB, and re-executes the write operations after resizing. With this, you do not have to make any estimates of database size, the databases automatically grow as needed (as you would expect from a database!)

`lmdb-store` is designed for synchronous reads, and asynchronous writes. In idiomatic NodeJS code, I/O operations are performed asynchronously. `lmdb-store` observes this design pattern; because LMDB is a memory-mapped database, read operations do not use any I/O (other than the slight possibility of a page fault), and can almost always be performed faster than Node's event queue callbacks can even execute, and it is easier to for code get instant synchronous values from reads. On the otherhand, in default mode with sync'ed/flushed transactions, write operations do involve I/O, and furthermore can achieve vastly higher throughput by batching operations. The entire transaction of batch operation are performed in a separate thread. Consequently, `lmdb-store` is designed for writes to go through this batching process and return a simple promise that resolves once the write is completed and flushed to disk.

LMDB supports multiple modes of transactions, including disabling of file sync'ing (noSync), which makes transaction commits much faster. We _highly_ discourage turning off sync'ing as it leaves the database prone to data corruption. With the default sync'ing enabled, LMDB has a crash-proof design; a machine can be turned off at any point, and data can only be corrupted if the written data is actually corrupted/changed. This does make transactions slower. However, by batching writes, when a database is under load, slower transactions just enable more writes per transaction, and lmdb-store is able to drive LMDB to achieve the same levels of throughput with safe sync'ed transactions as without, while still preserving the durability/safety of sync'ed transactions.

`lmdb-store` supports and encourages the use of conditional writes; this allows for atomic operations that are dependendent on previously read data, and most transactional types of operations can be written with an optimistic-locking based, atomic-conditional-write pattern.

## Usage
An lmdb-store instances is created with by using `open` export from the main module:
```
const { open } = require('lmdb-store');
// or
// import { open } from 'lmdb-store';
let myStore = open('my-store', {
	// any options go where, write-map mode is faster:
	useWritemap: true
});
```

Once you have a store the following methods are available:
### `store.get(key: Buffer, copy?: Function): Buffer`
Get the value at the specified key. The `key` must be a buffer, and the return value will either be a buffer if the entry exists, or `undefined` if the entry does not exist.

For typical usage, no `copy` function is needed, and returned buffer will be a safe copy of the data from the database. However, some optimizations can be performed (such as decompression) by using using the `copy` argument, in which case a `buffer` as a reference directly to the shared, memory-mapped data in the database will provided to the `copy` function and it will be responsible for copying the data (or decompressing directly from the db). The shared memory reference provided to the `copy` function is valid until a future writes or until a `remap` event, after which accessing the data will either be non-deterministic (after writes), or will seg-fault after a remap.

### `store.put(key: Buffer, value: Buffer, ifValue?: Buffer): Promise<boolean>`
This will set the provided value at the specified key. If the `ifValue` parameter is set, the put will only occur if the existing value at the provided key matches the value provided by `ifValue` at the instace the commit occurs (LMDB commits are atomic by default). If the `ifValue` parameter is not set, the put will occur regardless of the previous value.

This operation will be enqueued to be written in a batch transaction. Any other operations that occur within a certain timeframe (20ms by default) will also occur in the same transaction. This will return a promise for the completion of the put. The promise will resolve once the transaction has finished committing. The resolved value of the promise will be `true` if the `put` was successful, and `false` if the put did not occur due to the `ifValue` not matching at the time of the commit.

### `store.remove(key: Buffer, ifValue?: Buffer): Promise<boolean>`
This will delete the entry at the specified key. This functions like `put`, with the same optional conditional value. This is batched along with put operations, and returns a promise indicating the success of the operation.

### `store.putSync(key: Buffer, value: Buffer, ifValue?: Buffer): boolean`
This will set the provided value at the specified key, but will do so synchronously. If this is called inside of a synchronous transaction, this put will be added to the current transaction. If not, a transaction will be started, the put will be executed, and the transaction will be committed, and then the function will return. We do not recommend this be used for any high-frequency operations as it can be vastly slower (for the main JS thread) than the `put` operation (usually takes multiple milliseconds).

### `store.removeSync(key: Buffer, ifValue?: Buffer): boolean`
This will delete the entry at the specified key. This functions like `putSync`, providing synchronous entry deletion.

### store.transaction(execute: Function)
This will begin synchronous transaction, execute the provided function, and then commit the transaction. The provided function can perform `get`s, `putSync`s, and `removeSync`s within the transaction, and the result will be committed.

### `getRange(options: { start?: key, end?: key, reverse?: boolean}): Iterable<{ key: Buffer, value: Buffer }>`
This starts a cursor-based query of a range of data in the database, returning an iterable that also has `map`, `filter`, and `forEach` methods. The returned cursor/query is lazy, and retrieves data _as_ iteration takes place, so a large range could specified without forcing all the entries to be read and loaded in memory upfront, and one can exit out of the loop without traversing the whole range in the database. The query is iterable, we can use it directly in a for-of:
```
for (let { key, value } of db.getRange({ start, end })) {
	// for each key-value pair in the given range
}
```
Or we can use the provided methods:
```
db.getRange({ start, end })
	.filter(({ key, value }) => test(key))
	.forEach(({ key, value }) => {
		// for each key-value pair in the given range that matched the filter
	})
```
Note that `map` and `filter` are also lazy, they will only be executed once their returned iterable is iterated or `forEach` is called on it. The `map` and `filter` functions also support async/promise-based functions, and can create async iterable if the callback functions execute asynchronously (return a promise).

## Events

The `lmdb-store` instance is an <a href="https://nodejs.org/dist/latest-v11.x/docs/api/events.html#events_class_eventemitter">EventEmitter</a>, allowing application to listen to database events. There are two events:

`remap` - This event is fired before a database is resized, and the memory-map is remapped. If any data has been read using `get` with a `copy` function prior to this event, that data/buffer provided to `copy` _must_ not be accessed after this event, or it will cause a segmentation fault and your program will exit (this is non-recoverable).

`beforecommit` - This event is fired before a batched operation begins to start transaction to write all queued writes to the database. The callback function can perform additional (asynchronous) writes (`put` and `remove`) and they will be included in the transaction about to be performed (this can be useful for updating a global version stamp based on all previous writes, for example).

## License

`lmdb-store` is licensed under the terms of the MIT license.

## Related Projects

`node-lmdb` 