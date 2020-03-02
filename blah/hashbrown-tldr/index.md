% Swisstable, a Quick and Dirty Description

<span class="author">Alexis Beingessner</span>

<span class="date">July 27th, 2019</span>

Ok here's how to implement Swisstable as simply as possible, with a bunch of random notes. I'm assuming you're familiar with the basics of an open-addressing hashmap implementation, and will be skipping over all those details. If you want to brush up on those issues, you can read my primer on [the old Rust Robin Hood implementation](https://gankra.github.io/blah/robinhood-part-1/).

High level concepts to keep in mind:

* open-addressing
* searches in parallel using SIMD
* first-come-first-serve collision resolution
* chunked (SIMD) triangular (quadratic-ish) probing
* tombstones to avoid backshifts

Bit fiddling tricks will be omitted because they're not that interesting, and I made sure they were really well documented [in hashbrown](https://github.com/rust-lang/hashbrown/tree/master/src/raw) (the Rust impl), so you can just crib the implementations from there (seriously, it's only 100 lines of code, mostly comments). So if I ever say "do X in parallel (func_name)" that's your cue to check the parallel bit tricks implementation for func_name and its detailed documentation.

Just so you can get your bearings in that implementation:

* mod.rs - the actual hashtable impl
* bitmask.rs - a simple fixed-size bitvector impl; the result of parallel queries
* generic.rs - the software fallback impl of our parallel bit tricks
* sse2.rs - the sse2 impl of our parallel bit tricks

The hashtable has the parallel bit-trick details abstracted away by the generic and sse2 modules (impl picked at compile time).


# Memory Layout

A table's layout is defined by:

* n buckets (power of 2, for classic modulo trick reasons)
* a static Group WIDTH (16 bytes for SSE2; usize for software fallback; ARM always uses fallback because NEON is too high latency for our usecase)

A table is just a single allocation containing:

* an array of n+WIDTH "control" bytes
* an array of n keys of type T (your keys can be T=(K, V) to make it a proper map; you don't want to unzip them into an array of K and an array of V since accessing a key is highly correlated with accessing its value)

Our buckets are like `Option<T>`, where the control bytes are the Some/None tag, but having a whole byte means we can pack extra metadata in there (more on that later). We have an extra WIDTH control bytes on the end that mirrors the first WIDTH bytes so that we don't have to worry about how to partially wrap a SIMD access that should otherwise wrap around to the start of the array. However when `n < WIDTH`, this mirroring is a bit subtle. See the excellent documentation in set_ctrl for how this works. TL;DR - the mirror is right-justified, so you will have a bunch of empty buckets between the real buckets and the mirror ones.

Your outlined table metadata is:

* `bucket_mask: usize` (= n - 1)
* `ctrl: *u8` (points at the control bytes array; which is also the start of the alloc)
* `data: *T` (points at the start of the keys array, yes recomputing this is too slow; dangles in the empty case)
* `growth_left: usize` (number of empty spaces we can tolerate filling; tombstones deplete this without affecting len, so this isn't just a function of len, bucket_mask, and load-factor)
* `len: usize` (number of actual values stored in the map)

I tried storing the size/capacity/etc metadata in the same allocation, but that requires more branches to handle the static empty map, and I couldn't find any performance wins from making a HashMap thinner.

Load-factor is a tunable parameter, all that matters is that you make sure there is always at least one empty bucket (feel free to crib the implementation).




# Control Bytes and Hashes

A control byte has one of 3 values:

* `0b1111_1111` = EMPTY
* `0b1000_0000` = DELETED (tombstone)
* `0b0xxx_xxxx` = FULL (x is a hash fragment, more on that below)

This representation will be useful for us when doing cute bit tricks. Note that we can tell if a bucket contains a key or not by just looking at the top bit.

There are two operations we perform with a hash (I hate these names but these are the ones you'll see in the implementations):

* h1 - the whole hash truncated to a usize, used as an index into our array (apply modulo arithmetic to it)
* h2 - the top 7 bits of the hash, in FULL control byte format (see above); note that this will motivate you to have a hash function with good, uhhhhh, "entropy" in the high bits

When we insert a key into the table, we store its h2 value in the associated control byte. When searching the table we can compare our search key's h2 to each control byte. If they don't match, then we know that we don't have to look at the key stored there at all. With a good hash function we only have \~1/128 odds of a false positive here, so we almost never look at keys except for exactly equal ones! (We use the high bits for h2 so that they'll be uncorrelated with the index h1, which is the log(n) lowest bits.)



# Searching and Probing

Note that in general our SIMD accesses won't be properly aligned, so be sure to use the unaligned load/store functions. You can only use aligned ops when walking from the start of the array (iterator/realloc-ish stuff).

So, searching:

1. start at bucket h1 (mod n)
2. load the Group of bytes starting at the current bucket
3. search the Group for your key's h2 in parallel (match_byte)
3. for each match, check if the keys are equal, return if found
4. search the Group for an EMPTY bucket in parallel (match_empty)
5. if there was any matches, return false
6. otherwise (every entry was FULL or DELETED), probe (mod n) and GOTO 2

Probing is done by incrementing the current bucket by a triangularly increasing multiple of Groups: jump by 1 more group every time. So first we jump by 1 group (meaning we just continue our linear scan), then 2 groups (skipping over 1 group), then 3 groups (skipping over 2 groups), and so on.

Interestingly, this pattern perfectly lines up with our power-of-two size such that we will visit every single bucket exactly once without any repeats (searching is therefore guaranteed to terminate as we always have at least one EMPTY bucket).

Also note that our non-linear probing strategy makes us fairly robust against weird degenerate collision chains that can make us accidentally quadratic (Hash DoS). Also **also** note that we expect to almost never actually probe, since that's WIDTH (8-16) non-EMPTY buckets we need to fail to find our key in.




# Insertion

Insertion is done in two passes

* Pass 1: just run the search algorithm. If it returns a result, then overwrite that and return.
* Pass 2: actual insertion (now knowing the key isn't present, we search for EMPTY/DELETED).

So, actual insertion:

1. start at bucket h1 (mod n)
2. load the Group of bytes starting at the current bucket
3. search the Group for EMPTY or DELETED in parallel (match_empty_or_deleted)
4. if there was no match (unlikely), probe and GOTO 2
5. otherwise, get the first match and enter the SMALL TABLE NASTY CORNER CASE ZONE
6. check that the match (mod n) isn't FULL.

This can happen for small (n < WIDTH) tables, because there are fake EMPTY bytes between us and the mirror bytes. If it does happen, then we know: we're a tiny table that fits in a Group, and that the guaranteed-to-exist empty bucket wasn't anywhere between h1 and the end of the table.

7. if it *wasn't* FULL, return that location for insertion
8. if it *was* FULL, load the (aligned!) Group at the start of the table
9. search the Group for EMPTY or DELETED in parallel (match_empty_or_deleted)
10. return the first location for insertion (guaranteed to exist!)



# Deletion

The search routine for deletion is really easy: just run the search algorithm. There you found the thing to remove. The tricky bit is the cleanup.

So we saw in the searching algorithm that our cue to probe is an entire contiguous Group of FULL/DELETED entries. So if the entry we want to delete is in within such a contiguous Group, setting it to EMPTY may "break" the search path for any key that probed over it. So we need to check if that's the case, and if so, replace ourselves with DELETED. To put that another way: we need to find the nearest EMPTYs to our left and to our right, and make sure they're less than WIDTH apart.

Note how extreme that is: we only ever insert a tombstone if the entry we want to delete is in a pack of WIDTH non-EMPTY buckets. Under reasonable load factors that's very unlikely!

So, removal:

1. start at the removal bucket
2. load the Group of bytes starting at that bucket
3. search the Group for EMPTY in parallel (match_empty), and record the first empty found ("end")
4. load the Group of bytes *before* that bucket
5. search the Group for EMPTY in parallel (match_empty), and record the last empty found ("start")
6. if the distance between start and end is >= WIDTH, set the bucket to DELETED
7. otherwise, set the bucket to EMPTY

(implementation note, 3, 5, and 6 is just `a.leading_zeroes() + b.trailing_zeroes() >= WIDTH`, although be careful which is which in your implementation, lots of negations and endianess confusion to be had!)



# The Empty Singleton

You don't want to allocate for empty tables, so just statically allocate an Group-aligned(!) array of `[EMPTY; WIDTH]` and point your empty collection at it. If you implemented everything right, it should just workout because you'll load those WIDTH bytes into a Group, see it's empty, and always conclude there's nothing to do. When you need to actually insert, your load factor should tell you to resize right away. You mostly only need to add extra guards in places like `clear` and whenever you would realloc/free the table (all very slow paths, so no concern).

There's a minor hard-to-hit footgun here if your implementation somehow associates your pointer into the control bytes with the type of your keys. If you do this, the empty singleton may be under-aligned or you may run afoul of TBAA-style things. The fact that we hold separate pointers for keys and ctrls makes it pretty easy to avoid this. Just keep the ctrl pointer a boring untyped pointer-to-bytes.



# Extra Notes

* You need your hash function to resize the array, as you need to recompute everyone's h1 value, which you didn't store. This means that swisstable will take a nasty perf hit if you have a very expensive hash function. This also makes it harder to use external state for hashing (e.g. an interner the map doesn't know about), as more "random" operations will need that state passed in just in case.

* There's a ton of cute tricks you can do on resizes, but they're not super interesting to explain (read the code)

* The control byte mirror is really just implemented by always setting two ctrls unconditionally (locations, of course, computed with cute tricks). Most of the time you just write the same byte to the same location twice in a row, so not a huge perf issue.

* There's a bunch of random opportunities to do some operation branchless by doing a cute bitmask trick (e.g. updating `growth_left` by incrementing/decrementing based on the top bit of the ctrl)

* There's a lot of places where you could use unlikely/likely intrinsics, but we got really mixed results. Sometimes it helped codegen, sometimes it hurt it. Compilers.
