# Tungsten: How Spark Stopped Trusting the JVM

This is the story of Project Tungsten—Spark's internal initiative to stop relying on Java objects for data and start managing memory and computation in ways the JVM was never designed for. Tungsten is why Spark SQL can be competitive with hand-tuned C++ for certain workloads, why garbage collection is no longer the dominant cost it once was, and why the physical representation of a row inside a running query looks nothing like a Java object.

---

## The problem with Java objects

Every value in a JVM program is an **object** (or a primitive boxed into an object). An object has a header (typically 16 bytes on a 64-bit JVM) before any actual data. A `String` is not just its characters—it's an object pointing to a `char[]` array, which is another object. A `java.lang.Long` wrapping a single 8-byte number costs 24 bytes.

This means that for a table of one billion rows with a few columns each, the on-heap representation is a sea of tiny objects with headers, pointer indirections, and lots of wasted space. The **garbage collector** has to trace all of those objects to determine which are live. GC pauses scale with the number of live objects, not just their total size.

> **Think of Java's object model like a city where every item, no matter how small, must have its own house, address, and registered postal code.** A single button has its own house; a single screw has its own house. The city planner (garbage collector) has to inspect every house periodically to see if it's still occupied. For a billion items—one billion houses—the inspection takes hours. Tungsten is the industrial warehouse: all items in one building, located by their position on a shelf. The warehouse inspector walks one building, not a billion houses.

When Spark is running an aggregation over 10 GB of data—all of it in memory as Java objects—GC becomes the bottleneck. The CPU spends as much time in collection as in actual computation. Tungsten was built to address this.

---

## The UnsafeRow: a row is just bytes

The foundation of Tungsten is `UnsafeRow`, Spark's internal binary row format. An `UnsafeRow` is not a Java object with fields. It is a **contiguous region of bytes**, accessed directly via `sun.misc.Unsafe` (a JVM API that allows raw pointer arithmetic and off-heap memory access, bypassing normal Java safety checks).

The layout of an UnsafeRow is predictable. The first section is a **null bitmap**: one bit per field, indicating whether each field is null. Then comes a **fixed-width values section**: 8 bytes per field. For fixed-width types (integers, longs, doubles, booleans), the value is stored directly in those 8 bytes. For variable-width types (strings, arrays, maps), those 8 bytes store an offset and a length pointing into a **variable-length data section** at the end.

The whole thing is one contiguous allocation—no nested objects, no pointer chasing through the heap.

> **An UnsafeRow is like a spreadsheet row stored in a zip file.** You know that columns 1–4 are at bytes 0–31, column 5 is at byte 32, and the variable-length comment field is at whatever offset is stored at byte 40. To read field 3, you jump to byte 24 and read 8 bytes. No pointers to chase, no objects to open. Compare this to Java's object model, which is like a nested set of locked boxes where each box contains a key to the next box.

When Spark needs to read field 3 of an UnsafeRow, it computes the byte offset (known at compile time based on the schema), reads 8 bytes from that offset, and either interprets them as the value directly or uses them to find the variable-length data. One or two memory reads, no GC.

---

## Off-heap memory: escaping the collector entirely

UnsafeRows can live **on-heap** (in a Java byte array) or **off-heap** (in native memory allocated via `Unsafe.allocateMemory`, completely invisible to the GC).

Off-heap storage is the more radical option. When rows live off-heap, the garbage collector sees no references to them at all—there are no Java objects to trace. GC pauses drop dramatically regardless of how much data is live, because from the GC's perspective, hardly anything is alive.

> **Off-heap memory is like a storage unit you rent outside your house.** The tidying service (garbage collector) only cleans your house—the storage unit is completely invisible to it. You manage the storage unit yourself; it won't clean itself up automatically. The house stays tidy regardless of how much stuff is in the unit.

The tradeoff is that Spark must track that memory itself: it knows how many bytes it has allocated, and it is responsible for freeing them when done. Bugs in memory accounting can cause native memory leaks. So off-heap is a more brittle mode, but for memory-intensive workloads the GC elimination is often worth it.

---

## Cache-friendly layout: making the CPU happy

Modern CPUs don't read memory one word at a time. They read **cache lines**—64 bytes at a time. If your data is laid out so that sequential accesses jump around randomly in memory, you get cache misses on every access: the CPU spends most of its time waiting for memory, not computing.

The UnsafeRow format is **cache-friendly by design**. Because a row is one contiguous region, reading all the fields of a single row accesses one or two cache lines. When Spark processes rows in a tight loop, each iteration reads the next row sequentially from a buffer of contiguous UnsafeRows. Sequential access patterns mean the prefetcher can predict what memory will be needed next and load it before the CPU asks.

> **Cache-friendly access is like reading a book normally, page by page.** Your eye moves sequentially; the next word is always right next to the last. Java's pointer-chasing model is like reading a mystery novel where each clue sends you to a random page—you're constantly flipping back and forth, losing your place. Sequential access is fast; random jumping is slow.

Contrast this with the Java object model, where a row is a Java object pointing to field objects pointing to array objects. Reading all fields of a row chases three to five pointers, each potentially a cache miss.

---

## Memory management without the GC: Tungsten's memory manager

Tungsten's memory model organizes memory into **pages** (large contiguous allocations). Each page is either a Java byte array (on-heap) or a native allocation (off-heap). Pages are tracked by a `TaskMemoryManager` per task, which knows how many bytes of each type are in use.

When a sort buffer, aggregation hash map, or join build side needs memory, it asks the `TaskMemoryManager` for a page. If memory is available, it gets a page. If not, it may **spill to disk**: serialize the data in its current pages, write it to local storage, and release the pages. Spilling is fast because the data is already in a compact binary format—writing it to disk is essentially a memcpy.

---

## Binary operations: sorting, hashing, and comparing without deserialization

One of the most important properties of the binary format is that many operations can be performed on the bytes **without deserializing them back to Java objects**.

For **sorting**, Tungsten uses a **radix sort** on binary keys. If you're sorting by a single long column, the sort key is already 8 bytes in a known location. Radix sort needs no comparisons—it reads bytes and places records in buckets—so it runs in linear time.

For **hashing** (used in hash joins and aggregations), the hash function runs directly on the bytes of the key fields. No `hashCode()` call on a Java object—just a few XOR and multiply operations on raw bytes.

For **comparisons** (used in sort-merge joins), Tungsten can compare two rows field by field using simple byte reads and integer comparisons, without constructing any Java objects.

> **Binary operations on UnsafeRows are like a librarian who sorts books by their barcode number.** They don't need to open the book, read the title, understand the content, or know the author's biography. They scan the barcode (read the key bytes) and sort by number. Sorting doesn't require understanding—it just requires the right bytes in the right place.

---

## Tungsten and whole-stage codegen: a partnership

Tungsten's binary format and whole-stage codegen are designed together. Codegen generates Java code that directly reads fields from UnsafeRows at fixed byte offsets, bypasses the Volcano model's virtual dispatch, and operates on primitives rather than boxed objects. The generated code knows the schema at compile time, so it can hardcode the byte offsets for every field access.

Without Tungsten, codegen would still help (no virtual dispatch), but it would still box values and allocate row objects. Without codegen, Tungsten would still help (compact memory, no GC), but the per-row method calls would still be expensive. Together, they eliminate both the per-row allocation cost and the per-row dispatch cost.

---

## Bringing it together

Tungsten replaces Java objects with **binary rows**: contiguous byte regions with a fixed header, a fixed-width section for values, and a variable-length tail for strings and collections. These rows can live **on-heap** (as byte arrays, one object per buffer of many rows) or **off-heap** (in native memory the GC never sees). The binary layout is **cache-friendly**: sequential row access is sequential memory access. **Sorting, hashing, and comparing** are done directly on bytes, without deserializing to Java objects. Memory is managed in pages by a per-task memory manager, with spill-to-disk when tasks exceed their memory budget. Together with **whole-stage codegen**, Tungsten reduces the hot path of an analytic query to a tight loop over primitive values in contiguous memory—something a JIT compiler can optimize to near-native efficiency. So Tungsten is how Spark, a JVM application, achieves the kind of performance we usually only expect from systems written in C or C++.
