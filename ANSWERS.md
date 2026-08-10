# Week 1

## Day 1 — Memtable

### Correctness and Concurrency

#### 1. Why doesn't the memtable provide a delete API?

Because it would violate the append-only property of a memtable, that's why a deletion is emulated as an insert of an empty value for a specific key. Also, if you remove a key from the memtable, the old values in immutable memtables or sstables would resurrect, because there is nothing that says those are stale values if you delete the latest value, that's why we do an insert of an empty value for a key.

#### 2. Does it make sense for a memtable to store every write instead of only the latest version of a key? For example, suppose a user writes a -> 1, a -> 2, and a -> 3 to the same memtable.

No, the current implementation of the Skiplist we are using doesn't store every write of a key, it actually overwrites the values. Storing the latest write is good enough for reads, and storing anything intermediate just wastes memory

#### 3. Why do we need a combination of state and state_lock? Can we only use state.read() and state.write()?
(not 100% sure about my answer on this one)

We use `state`, a RwLock, because the Skiplist is a lock free data structure, so we can only acquire read locks, which a RwLock lets multiple threads acquire, and insert with enormous throughput. `state_lock` is a mutex that comes into play only when we need to freeze the memtable to avoid another thread to freeze a newly created memtable. Using only `state.write()` would serialize all writes to the memtable, killing concurrency and throughput. 


#### 4. Construct the smallest example in which probing memtables in the wrong order returns a stale value. Then construct one in which it resurrects a deleted value.

This is simple, imagine we have a current memtable, let's call it cur_memtable, and two frozen memtables (aka 2 items in imm_memtables),where the latest is called imm_memtable_a and oldest is called imm_memtable_b. We will use this premise for the two different scenarios 

- The key we are looking for is key `a`, and it's present as `a->3`, `a->2`, `a->1` in imm_memtable_b, imm_memtable_a and cur_memtable, respectively. It's trivial to see that looking at any of the immutable memtables will return a stale value. Even if there wasn't any key value pair for key `a` in cur_memtable, starting by the oldest immutable memtable would also produce a stale value 

- The same can be said if the value in cur_memtable for the previous scenario was `a->empty`, because looking at the immutable memtables first would also resurrect a deleted value

#### 5. After a memtable is frozen, could a thread that still holds an old LSM-state snapshot write to that now-immutable memtable? How does your solution prevent this?

The answer is no, a thread can't hold a snapshot to that now immutable memtable. First, we acquire a `state_lock`, to serialize changes to the LsmStorageState, then we acquire a write lock over the memtable, which guarantees that we don't have any read locks, preventing other threads from having a snapshot over a memtable that is about to be frozen.

We also don't actually clone `Arc<LsmStorageState>`, which helps avoid this issue, but I don't think we have any guarantees against this at code/compiler level.

#### 6. In several places, you might acquire a state read lock, release it, and then acquire a write lock. The two operations may occur in different functions that call one another. How does this differ from directly upgrading a read lock to a write lock? Is an upgrade necessary, and what does it cost?

After a bit of researching, I found that only parking_lot RwLock allows to upgrade a read lock to write lock, while this isn't available in the stdlib RwLock. ALso, parking_lot only allows you to one thread to hold a upgradable-read slot, which might hurt throughput a lot, because they will serialize, which isn't necessary for a plain read. The most common case of the "acquire a state read lock, release it, and then acquire a write lock" case I can think of right now is when you do a put (acquire read lock and then release it) and then freeze the memtable (acquire write lock). The main thing here is that I don't believe the complexity of adding this would be worth it, because I can't see any advantages, it will still block when upgrading to a write, it might work as mutex and not a actual RwLock because of the serialization, and the upgrade can fail, which is another thing we need to deal too. I don't believe it's necessary because it would hinder throughput because most callers won't freeze the memtable, and we are using the cheapest lock possible.


### Performance and Design

#### 7. Could an LSM tree use other data structures for its memtable? What are the advantages and disadvantages of a skiplist?

Yes, I know that Cassandra uses something else for their memtables, but usually if they aren't using a skiplist, the closest data structure that I can think of that fits the bill is a tree, with the whole node thing. Most trees, like B trees, AVL trees or red-black trees have great cache locality, but need locking and rebalacing. We could also use a hash map, which has concurrent implementations, but it lacks range scans.

Skiplists are good because they are lock-free, doesn't need to balance itself like a tree would, which make the great for append-only workloads. They suck at cache locality though.

#### 8. Is the memtable's memory layout efficient? Does it have good data locality? Consider how Bytes is implemented and stored in the skiplist. How could you optimize the memtable's layout?

No, because a Skiplist has horrible data locality because it is a non-contiguous heap allocated object, so a given key, we have a Bytes object for the key and another for the value, and a Skiplist node, all allocated in the heap and non-contiguous. I've read papers about how someone might fix this at the data structure layer, by implementing auxiliary data structures like trees (even if not replacing the entire skiplist with a tree data structure)


#### 9. Documentation check: Read parking_lot's RwLock fairness section. What might happen to readers waiting to acquire the lock when a writer is already waiting for the current readers to release it? How does eventual fairness differ from strict first-in, first-out service?


The writer might starve, because it a first-in, first-out kind of thing, acquiring a read lock is simple, and the readers will keep coming, starving the write lock. Eventual fairness guarantees that if a writer is waiting for too long, the readers will be blocked so the writer can be served.


## Day 2

<!-- Add questions here -->

## Day 3

<!-- Add questions here -->

## Day 4

<!-- Add questions here -->

## Day 5

<!-- Add questions here -->

## Day 6

<!-- Add questions here -->

## Day 7

<!-- Add questions here -->
