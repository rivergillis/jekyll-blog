---
layout: post
title:  "Demystifying malloc"
date:   2020-11-07 09:29:20 +0700
tags: [c, memory, programming]
description: A journey through the implementation of malloc.
---
It feels wrong to use a tool without knowing fully how it works. As programmers it is hard to accept that there is only so much that can fit into our noggins at once, but looking at code benchmarks or stack traces to see that some large amount of time is spent in some low-level C code always makes me wonder what is really going on in there. Maybe you're like me and occasionally start to use the 'Go to definition' IDE feature on standard libraries, and after a second or two of searching your window fills with scary underscores and #DEFINEs of things you didn't know existed, you think maybe this thing was auto-generated and no human would bother writing this header-file-hell. Unsatisfied you go back to whatever you were working on, no closer to understanding what's *really* going on down there.

For me I was always perplexed by `malloc`. It's a simple function, you ask for memory and it gives it to you. But *how* could a C function do that? What on Earth does it mean to *allocate* memory? Isn't it all there, sitting on the bus, just **waiting** for us to issue some good 'ol `MOV` instructions? Even worse, it is used *everywhere*. Even if you aren't using C there's a good chance you're using `malloc`, every time you create a new object in a language implemented in C, like `cpython` for instance. It isn't the only way to acquire memory, but it sure is a popular one. 

So let's take a look at `malloc`, it can't be that complicated right? Here's `malloc`:

```c
void* malloc(size_t size) {
  return sbrk(size);
}
```

From this you may be able to figure out what `sbrk` does. It gives us a chunk of memory. Specifically it allocates program heap memory of a given size and returns a pointer to it. What exactly is "it" that is being pointed to? Let's start printing stuff and find out.

```c
void* a = sbrk(0);
void* b = sbrk(100);
void* c = sbrk(0);
printf("[a: %p]\n[b: %p]\n[c: %p]\n", a, b, c);
```

```
> [a: 0x1065b4064]
  [b: 0x1065b4064]
  [c: 0x1065b40c8]
  ```

Looks like `b` is the same as `a` and `c` is `b + 100` bytes. So `sbrk` returns a pointer to whatever the last 'tip' of the program heap is, and if we give it a size in bytes it'll move that 'tip' up. Meaning if we `sbrk` ourselves that 100 byte chunk we can do whatever we want with it knowing that the next time we `sbrk` ourselves some more memory, it'll be 100 bytes farther along.

This may not be too satisfying, we've replaced one magical function with another. However, in this case `sbrk` is a system call. It's going to jump the CPU over to some assembly to execute (the instruction set implemented by your CPU is very likely to have a set of functions for interfacing with memory), at least now we're talking to the kernel  instead of wondering what's going in the the C standard library.

So that's `malloc`, simple right? Well judging from the length of this article you can probably deduce otherwise. There's more here, and for two reasons:

1. `sbrk` is absolutely ancient and super-deprecated. In fact if you run these snippets on macOS you're going to get tons of warnings (but hey, it still works!). It doesn't work with virtual memory and it isn't thread-safe. However, its API is very simple to use and `malloc` at one point in time very likely was implemented using `sbrk`.
2. This implementation of `malloc` is incorrect. The first reason why, which you may be able to guess, is that `sbrk` can fail. Memory is a finite resource.

According to `man sbrk`, the call can return -1 if it fails, but `malloc` is supposed to return NULL. This is fixed easily enough.

```c
void* malloc(size_t size) {
  void* chunk_start = sbrk(size);
  return chunk_start == (void*)-1 ? NULL : chunk_start;
}
```

One more thing. `malloc(0)` has special behavior, it needs to return `NULL` as well, otherwise you'd be able to get a pointer back from `malloc` that you didn't actually allocate, and that would be weird.

```c
void* malloc(size_t size) {
  if (size == 0) return NULL;
  void* chunk_start = sbrk(size);
  return chunk_start == (void*)-1 ? NULL : chunk_start;
}
```

Great, now the user can actually know whether their memory request was fulfilled. We're still missing something though, the result of our `malloc` doesn't work with `free`. What *is* `free` exactly?

According to `man free`, `free` will remove the allocation ("free"ing the space) from an input pointer that was previously returned from malloc. Now `sbrk` has a feature where when a negative input is passed to it, it will move the tip of the heap *down* instead of up. So effectively it allows us to push and pop from the program heap, because the heap is a stack and computer terminology is silly.

Unfortunately this isn't enough for us. Imagine a user does the following:

```c
void* a = malloc(500);
// We can now call sbrk(-500) to free a.
void* b = malloc(1000);
// But how do we free a from here?
```

This is the age-old problem of trying to delete something from the middle of the stack. We could pop everything off of the stack until we reach the memory we're trying to delete (storing it somewhere else, a disk for instance), then pop the item to delete, then push everything else back onto the stack. This would be miserably slow. We could also abandon the stack mentality and just use `memcpy` to copy over the old bytes. This would also be very slow, usually we expect `free` to take an insignificant amount of time to complete. In either of these cases, we've created a new problem: when shifting all of the old memory to utilize the newly free'd space, all of the pointers in the program refering to that old memory would be invalidated.

It looks like we're going to have take matters into our own hands. Maybe in the future we'll have more memory than we know what to do with and never free anything. Until then we'll need to do something clever. We have one thing going for us though, `malloc` always returns a pointer to *contiguous* memory. If we have a single "hole" of free'd memory in the heap large enough to use somewhere, we can use it. It's simply a matter, then, of us keeping track of the allocated chunks (and the "holes" created by `free`ing those chunks) ourselves.

Let's set up a general outline of what we want to accomplish.

```c
typedef struct chunk {
  size_t size;  // size of user-accessible memory.
  struct chunk* next;
  bool allocated;
} Chunk;

Chunk* heap;

void* chunk_data(Chunk* chunk) {
  // Adding 1 to a Chunk* will get us to the part of memory
  // directly after the fields.
  return chunk ? chunk+1 : NULL;
}

// Returns the Chunk corresponding to the chunk's data.
Chunk* chunk_metadata(void* ptr) {
  return ptr ? (Chunk*)ptr-1 : NULL;
}

 // TODO: Do the hard part.
Chunk* find_or_reserve_chunk(size_t size);

void* malloc(size_t size) {
  if (size == 0) return NULL;
  return chunk_data(find_or_reserve_chunk(size));
}

void free(void* ptr) {
  Chunk* chunk = chunk_metadata(ptr);
  if (!chunk) return;
  chunk->allocated = false;
}
```

We want some way to model the heap, so a linked list sounds simple enough. We could extend it to create an actual stack, but it isn't really needed here. The general idea is that for every chunk of memory allocated by `malloc`, the we store a few bytes (specifically `sizeof(Chunk)`) of metadata about that chunk right beforehand. We could store a pointer in Chunk to the actual user memory, but since the memory is contiguous we can easily compute where the metadata ends and the user data begins. `free`ing then becomes super easy, we can just get the metadata and mark that it's no longer allocated. The hard part is using that information.

```c
// Allocates a new chunk right after 'prev'.
Chunk* allocate_chunk(Chunk* prev, size_t size);

Chunk* find_or_reserve_chunk(size_t size) {
  Chunk* prev_chunk = NULL;

  // Initialize the heap if necessary.
  if (!heap) {
    heap = allocate_chunk(prev_chunk, size);
    return chunk_data(heap);
  }

  // Scan the heap for holes large enough for the chunk we want.
  Chunk* chunk = heap;
  while (chunk) {
    if (!chunk->allocated && chunk->size >= size) break;
    prev_chunk = chunk;
    chunk = chunk->next;
  }

  if (chunk) {
    chunk->allocated = true;
  } else {
    chunk = allocate_chunk(prev_chunk, size);
  }

  return chunk;
}
```

From here you can devise faster ways of doing this. The memory-speed tradeoff here is real, you can avoid scanning the heap every time by reserving a portion of the start of the heap for a hash map storing holes by size requirement, but then of course you've got less of the heap for the user. You also need to consider the size of this portion. You won't be able to grow it, since then you'd be invalidating user pointers.

If speed is less of a concern, you'd want to scan the entire heap rather than stopping at the first available hole. With the above implementation the user would often get back chunks of memory where the allocation is actually greater than they requested. If you scan the entire heap, you can look for the smallest hole that fits the requirements. Another way of doing this is to terminate the chunk to always fit the requested size, creating a new chunk for the leftover data. You'd have to make sure whatever leftover has enough room for the metadata fields.

This is why the data stored in the pointer returned from `malloc` is uninitiliazed, it may have been a chunk from some previously requested memory.

The last bit is where we actually build up the heap model:

```c
Chunk* allocate_chunk(Chunk* prev, size_t size) {
  Chunk* chunk = (Chunk*)sbrk(size + sizeof(Chunk));
  if (chunk == (Chunk*)-1) {
    return NULL;
  }

  if (prev) {
    prev->next = chunk;
  }

  chunk->size = size;
  chunk->allocated = true;
  chunk->next = NULL;
  return chunk;
}
```

We take in the last chunk in order to avoid needing to scan the heap again. We need to be careful about asking `sbrk` for enough memory, since we're also storing the `Chunk` metadata. Because `sbrk` returns -1 as a pointer on failure, we have to do some weird casting in order to generate the literal for comparison.

There are a few other memory allocation functions like `calloc` and `realloc`, but if you've gotten this far you can probably determine how to implement those in terms of `malloc` and `free`. A fun challenge would be figuring out how to implement `aligned_alloc`. There's also the issue of thread-safety, you can imagine things will go very poorly if two separate threads are operating on `heap`, a simple way to reconcile this is to protect it with a mutex.

Having written this I feel at ease once again, `malloc` makes sense and all is right in the world. The trick was just learning how to model the heap within the heap itself. This is a very basic and slow implementation, but it gets us up and running and can be used for accessing dynamic memory in programs, which is all the header files really ever promised us.