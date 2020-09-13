---
title: "The Cache Lectures"
date: 2020-09-12T12:21:45+02:00
draft: false
---
# The basics of cache, cache coherency, and "false sharing"

Based on
* ["What Every Programmer Should Know About Memory"](https://people.freebsd.org/~lstewart/articles/cpumemory.pdf), by U. Drepper,
* the [cache memory lectures](http://web.cecs.pdx.edu/~harry/videos-cache/) by Harry Porter (from Portland State University),
* and Scott Myers' 2014 talk ["Cpu Caches and Why You Care"](https://www.youtube.com/watch?v=WDIkqP4JbkE).

{{< hr >}}

In this video, we're going to talk about a cache-related slowdown phenomenon that can occur for parallell code called "false sharing". False sharing may cause significant slowdowns while at a first glance being hard to detect. But if you remember how cache works, and use ```perf``` like a boss, "false sharing" ain't got nothin' on you.

## Why cache?

Roughly speaking, there are two kinds of RAM. Static RAM (SRAM) and dynamic RAM (DRAM). DRAM is cheap, needs to continually refreshed, and is slower than SRAM, which is fast but expensive. Modern computers therefore have a RAM tradeoff -- they use a small amount of SRAM close to the CPU, and a larger amount of DRAM connected to the CPU via a bus.

The most common way of using the precious SRAM is to store recently used data or instructions -- to **cache** them. This turns out to be pretty effective, since programs often access the same area of memory repeatedly (spatial locality), and a lot of instructions are used repeatedly in rapid succession (temporal locality).

## Cache hierarchy

Let us recall what Martin told us about hardware architecture in week 2. We recall that CPU caches are divided into **levels**.

The layout could be something like the following (if you want to know for sure, you should **always** take a look at your CPU datasheet):

1. L1 cache -- directly connected to an individual core. Often split into L1i (instruction cache) and L1d (data cache).
2. L2 cache -- shared between cores.
3. L3 cache -- shared between all cores.

In rare cases, there is also an L4 cache (such as for some processors in the Broadwell and Haswell architectures). Should you be in a spot where you have to code in such a way that you want to effectively utilize an L4 cache, you're well beyond the scope of this course, and then I'll leave to your own devices. I'll also not talk so much about L1i, because frankly, I'm still learning about it.

For the visual learners among you, it might be beneficial to think of the above cache hierarchy in terms of the following picture:

{{< image src="../cache_layout.png" >}}

In this picture, we have two separate processors on the same bus. This is of course not typical for a desktop computer, but is the case for Gantenbein, which we recall from the output of ```lscpu``` is dual socket.

```
Architecture:                    x86_64
CPU op-mode(s):                  32-bit, 64-bit
Byte Order:                      Little Endian
Address sizes:                   46 bits physical, 48 bits virtual
CPU(s):                          112
On-line CPU(s) list:             0-111
Thread(s) per core:              2
Core(s) per socket:              28
Socket(s):                       2
NUMA node(s):                    2
Vendor ID:                       GenuineIntel
CPU family:                      6
Model:                           85
Model name:                      Intel(R) Xeon(R) Platinum 8180 CPU @ 2.50GHz
Stepping:                        4
CPU MHz:                         1000.079
BogoMIPS:                        5000.00
Virtualization:                  VT-x
L1d cache:                       1.8 MiB
L1i cache:                       1.8 MiB
L2 cache:                        56 MiB
L3 cache:                        77 MiB
[...]
```

However, on Gantenbein's two Intel Xeon Platinum 8180, and on [many modern Intel CPUs](https://stackoverflow.com/questions/944966/how-are-cache-memories-shared-in-multicore-intel-cpus), only L3 is shared between all cores and the L1 and L2 caches are private (not shared).

## Cache lines

We programmers are never in direct control of cache. Whenever the CPU reads or writes data, it's going to store the data (and a neighborhood of it) in cache.<cite>[^1]</cite> All we can do is to place the data in a cache-friendly way.

Whenever the CPU needs some data in memory, it's going to search through the cache by level for that data first (going from L1d, to L2, to L3). The way this is accomplished is that the address of the data is associated with a **tag** that the cache memory uses to locate the data in the cache. So, if the CPU wants to access data at address \\(X\\), it'll first search through the cache for the tag associated to \\(X\\). Since the cache is small, this will be faster than reading from main memory, given of course that we find the data in the cache (in cache lingo -- if we get a *cache hit*). If we don't find a match (in cache lingo -- a *cache miss*), then we use the tag to place the data (and a neighborhood of it), in the cache.<cite>[^4]</cite>

Since it's likely that we'll use neighboring memory together<cite>[^2]</cite>, and since main memory (to be more specific -- DRAM) is faster when reading a lot of data in a row (because of capacitors and such), we don't read just single bytes into cache. Rather, we read contiguous chunks of **64 bytes** of main memory (in cache lingo, *blocks*) into cache.<cite>[^3]</cite> Cache is indexed in terms of these chunks, which henceforth are referred to by their proper name **cache lines**.

The following picture might help you to remember:

{{< image src="../mem_vs_cache.png" >}}

Keep in mind however that the above picture is just a heuristic. In reality, what you see when you code are virtual memory addresses, which are mapped to physical addresses by the OS kernel and the CPU. All you need to understand as a (Linux) programmer is the concept of a process.

Now we get into a rather hairy area of caching -- how to associate memory addresses to cache lines in a way that makes it fast to find data in cache, and to place data in cache.

## Addresses to cache lines

In this section we'll talk about fully associative caches, direct mapped caches, and set-associative caches. The former two are special cases of the latter.

Henceforth, let us consider memory addresses as being divided up to into 3 pieces, as in the following picture:

{{< image src="../tag_set_offset.png" >}}

The number of tag bits and set bits will vary depending on the cache implementation. The number of offset bits will depend on the size of the cache line.

The tag, set, and offset bits, are used by the cache to look for a match. The tag and set bits are used to find (if it exists) a cache line that contains the (cached copy of the) byte the address is pointing to. The offset bits is used by the cache to select one of the bytes in cache line. So if we have a 64 byte cache line, we need \\(\log_2(64)=6\\) offset bits.

### Fully associative caches

In a fully associative cache, we don't use any set bits, so we only have to concern ourselves with the tag and the offset bits. Fully associative caches work a bit like "associative arrays" (also known as "maps" or "dictionaries"). The keys in the array are the tags, and the values are the cache lines. In addition to this, there is some logic to use the offset bits to select the appropriate byte in the cache line. Unlike the associative arrays you've encountered when programming, a fully associative array has a very limited amount of keys, and you can't just add new keys at will. Instead, if there's no match and you want to place data in cache, you have to decide to **evict** an already existing line. There are several different algorithms (cache replacement policies) to decide on which cache line to evict, e. g. least recently used, first in first out, last in first out, and so on. The interested reader is advised to consult [this Wikipedia page](https://en.wikipedia.org/wiki/Cache_replacement_policies). If it turns that the cache line that is going to be evicted has been modified (in cache lingo, it's *dirty*), then it needs to be written back to memory first.

> **REMARK:** It's important to note however, that in a fully associative cache, there's no hardware restriction on which cache line a main memory block is put into. This will not be the same for direct-mapped caches, as we shall soon see.

Here's a way of thinking of it in Julia:

```julia
function match(cache::Dict{BitVector, Vector{Char}}, addr::BitVector)
  tag = addr[7:64]
  offset = addr[1:6]
  cacheline = FAC(tag)
  if !isnothing(cacheline)
    return cacheline[Int(offset)]
  else
    #evict and replace
  end
end
```

Let me draw a simple picture for you of some idealized matching logic of a fully associative cache with 4 lines and 64 bytes per line -- a 256 byte cache.

{{< image src="../fully_assoc.png" >}}

In this picture `Cmp` is a component that returns `1` if all bits match, otherwise it returns `0`. The components `Sel` returns the byte indicated by the offset in the data coming from the right input if the left input is `1`, otherwise it returns the byte `0x00`.

Obviously, more logic is required to implement the (re)placement policy, but we'll that for another course. If you can get a hold of some gates, it'd be an interesting Shannon-esque exercise.

Fully associative caches seem great in theory, but if you want to have a lot of cache lines, it'll be pretty hard to implement in hardware. But for some special caches, like the translation lookaside buffer (TLB, used in conjunction with the translation between virtual addresses used by processes, to physical addresses), fully associative caches are used.

### Direct mapped caches

In a direct-mapped cache, you are no longer free to put a main memory block in whichever cache line you want. Instead, this will be determined by the set bits. That's right, for direct-mapped caches, we use all three of the tag, set, and offset fields. In a direct-mapped cache, we don't have compare the tag bits of the address with a whole bunch of cache line tags, and this makes it easier to build than a fully associative cache (just use a multiplexer).

To see how we use the fields, let's look at an example of a 512 line, 64 byte/line cache. (This time I'm not drawing out any circuitry, as I don't think it helps particularly with the understanding.)

{{< image src="../direct_mapped.png" >}}

In the above example, when looking for a byte with address \\(A\\) in cache, we:

1. Send in \\(A\\) to cache.
2. Extract the set bits from \\(A\\).
3. Use the set bits to select the associated cache line.
4. Compare the tag bits of \\(A\\) with the tag on the cache line. If there's match, we've a cache hit, and if there's no match we've a cache miss. In the case of a miss, we proceed along the same lines as described in the previous section.

On a given cache line, say with set bits `0d123=0b001111011`, the possible tags are

>`0b0000000000000000000000000000000000000000000000000`

through to

>`0b1111111111111111111111111111111111111111111111111`

Hence the possible starting addresses of the memory block stored in the cache line are

* `0b0000000000000000000000000000000000000000000000000 001111011 000000 = 0d7872` -- associated with the main memory block with starting address \\(7872\\) and ending address \\(7935\\).
* `0b0000000000000000000000000000000000000000000000001 001111011 000000 = 0d40640` -- associated with the main memory block with starting address \\(40640=7872+2^{15}\\) and ending address \\(40703\\),
* `0d73408`
* and so on.

There is a drawback with direct mapped caches. In the above example, it is possible (although unlikely) that our program needs to access data at for example the main memory addresses and `0d7872` and `0d40644` in close succession. This data belongs to blocks that will be cached on the same line, and therefore we need to repeatedly evict the cache line. This situation has a large hit on performance and is referred to as [*cache thrashing*](https://en.wikipedia.org/wiki/Thrashing_(computer_science)). (See also Harry Porter's 6th video at 9:40.)

One way of avoiding issues such as cache thrashing is to use a compromise between fully associative, and direct-mapped caches: set-associative caches.

### Set-associative caches

Set-associative caches work by dividing up the cache lines into sets, where each set of cache lines work like a fully associative cache.

I'll draw you a picture again, and then I'll explain how it works. The picture depicts a 32 kilobyte 8-way associative cache with 64 bytes per line.

{{< image src="../set_assoc.png" >}}

This cache is called 8-way because every cache set contains 8 cache lines, and we see that

$$8\cdot 64\cdot 64\text{ B}=32768\text{ B}=32\text{ kB}$$

so that indeed the total size of the cache is 32 kilobyte, as we claimed.

Let's now describe how this cache works when looking for a byte with main memory address \\(A\\).

1. Send in \\(A\\) to the cache.
2. Extract the set bits from \\(A\\).
3. Use the set bits to select the associated cache set.
4. Compare the tag bits of \\(A\\) with the tags of the 8 cache lines in the set (like in a fully associative cache). If there's a cache hit, you're golden -- read your desired byte from the cache line using the offset bits. If there's a cache miss, follow the cache placement policy.

What's neat about a set-associative cache is that the likelihood of thrashing your cache is significantly reduced. There's no longer a single tag in the cache set (like with a direct mapped cache), and instead there are 8 (in our example).

Hence, we could store the bytes with addresses

> \\(A=\\) `0b0000000000000000000000000000000000000000000000000000 010111 000001`

and

> \\(B=\\) `0b0000000000000000000000000000000000000000000000000010 010111 000011`

in the same cache set (with index `0d23`).

### Concluding thoughts on cache layout

The above is only a very brief overview of cache layout, and there is far more to be learned if you want to really go far down the rabbit hole. However, for the purpose of understanding cache-related slowdown issues, it should be more than enough.

## Cache coherency -- the MESI protocol

Now we're finally ready to talk about cache-related slowdowns in a multi-core setting. In this setting, all cores should, in the words of Drepper, "see the same memory content at all times".
[^4]: As you'll see later, this is a heavily simplified picture, and is only really true for fully associative caches.
[^1]: As Drepper mentions, there are some places in memory that cannot be cached, but that's something for the OS-people.
[^2]: This is a bit of a "chicken or the egg" kind of situation. As programmers we learn that cache bets on spatial locality, and therefore we try to use contiguous blocks of memory, by choice. But CPU developers assume that our code uses memory contiguously and therefore design the CPUs to take advantage of this. Regardless, contiguous \\(\Leftrightarrow\\) good.
[^3]: On a 64-bit system, we can read 8 bytes (the word size) over the memory bus, and hence, filling up a cache line requires 8 transfers over the bus.
