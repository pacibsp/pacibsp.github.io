---
layout: post
title:  "Some thoughts on memory safety"
---

In this post I want to share a few thoughts on some more theoretical aspects of memory safety. These points aren't necessarily new, but I feel like they're sometimes underappreciated.

## Memory unsafety is a specific instance of a more general pattern of handle/object unsafety

Many patterns of memory corruption also occur in code that would otherwise be considered memory safe.

For example, a Rust program might use a `Vec` as storage for a set of objects and reference those objects in other parts of the program by using indexes that act as pseudo-pointers. If a slot is marked as free, reused for a new object, and then the stale index that semantically refers to the old object is used and inadvertently accesses the new object, that effectively creates a "logical UAF". The same pattern can also happen through APIs that return handles, for example with file descriptors. A program might close an fd, open a new file that gets assigned the same fd number, and then invalidly try to write to the old file using the old fd, corrupting the new file.

In both of these cases, the "UAF" is undoubtedly memory-safe. But the structural similarity to memory corruption UAFs is notable and feels important to me.

I've started describing the common structure between traditional memory unsafety and these similar-looking logic bugs as handle/object unsafety: You have objects, you refer to those objects through handles, and there's some translation scheme through which you can use a handle to access the corresponding object. Abstractly, it's theoretically possible to track the association between handles and objects perfectly, such that a handle could be made to only ever translate to the semantically-correct object. However, on real systems, practical constraints force the translation from handle to object to be imperfect in ways that allow a handle that is semantically tied to one object to be used to access some other object instead.

This highlights how memory-safe code can recreate known-bad patterns from the memory-unsafe world. For example, when Rust code uses indexes as pseudo-pointers, it effectively scrubs them of semantic information like lifetimes and borrows. Without that information, the compiler cannot enforce invariants to constrain the program's use of those pseudo-pointers to be semantically meaningful.

## Memory unsafety is relative to a particular layer in a stack of abstract machines

Consider taking a C program, compiling it to WebAssembly, and running it on a WASM VM. If the original C program has a memory unsafety bug, is the resulting execution of the program memory-unsafe?

My take is that you need to ask the question at a particular layer of the stack of abstract machines. Modern software and systems development lean heavily on the abstract machine concept, although I rarely see this discussed explicitly.

I like to think about programming languages as "abstract machine adaptors": they take one abstract machine (e.g. your underlying CPU) and they provide a new abstract machine in which it's easier to write programs (e.g. the C language). Compilation in this view is the act of taking a program written for one abstract machine and creating a new program that faithfully replicates some acceptable version of the original program's behavior on the underlying abstract machine. Program interpreters simulate an abstract machine dynamically. Even an operating system can be seen as an adaptor that takes a physical CPU conformant to the CPU vendor's ISA and provides a "userspace program" abstract machine which extends the ISA with conveniences like syscalls to allocate virtual memory and perform I/O.

In this framing, you could see the stack of abstract machines in the original example as follows:
* The physical CPU provides a "CPU ISA" abstract machine.
* The operating system plugs in to the CPU ISA abstract machine and provides an "operating system process" abstract machine capable of running userspace programs.
* The WASM VM is a userspace program that plugs in to the operating system process abstract machine and provides a "WASM" abstract machine.
* The C-to-WebAssembly compiler produces a program that plugs into the WASM abstract machine and provides a "C-language" abstract machine for which any C-language program can be written.
* And finally, the original buggy C program is written to run on the C-language abstract machine provided by that compiler.

Broken down like this, it becomes obvious that the buggy C program exhibits memory unsafety from the perspective of the C-language abstract machine, i.e. inside the WASM VM, but not from the perspective of the WASM abstract machine, i.e. within the operating system process implementing the WASM VM itself. While simulating memory corruption in the C program, the WASM VM respects all of its own memory safety invariants for its own allocator.

Thus, memory unsafety is in a sense a relative concept: it matters in which layer in the stack of abstract machines the unsafety is taking place.

## Memory unsafety matters because it violates local reasoning about state

This argument is well articulated in the excellent 2018 paper [The Meaning of Memory Safety](https://arxiv.org/pdf/1705.07354). I'm on the fence about whether local reasoning is a _definition_ for memory safety or an _explanation_ for why memory safety is the most problematic bug class in modern software. Either way, noninterference and local reasoning about state are crucial to a wholistic understanding of memory safety.

Intuitively, the basic idea is that a piece of code and corresponding state (i.e. memory) should only be able to influence what is "reachable" from that code/memory. For example, if you look at where an object gets allocated and trace through all of the code that touches that object, that should give you a wholistic understanding of the state of the object at any time. Local reasoning about state says that you don't need to look at some unrelated piece of code or memory to understand what state your object is in, because unrelated code/memory cannot interact with the object except through the code/memory that _does_ touch the object.

Memory corruption violates local reasoning about state, and that is the reason why memory corruption is so powerful for building exploits, as I tried to convey in my prior post.

Unfortunately there are some limitations with the paper's approach that make it hard to apply to real systems. The locality/noninterference arguments are proposed in the form of several frame theorems. If you try to apply the first frame theorem to show isolation in a program that uses an allocator wrapper, you'll quickly notice that the non-reachability precondition is not satisfied. This is a toy example, but it demonstrates how the paper's framework cannot help you reason about locality and noninterference when state isolation is enforced by invariants in the code (e.g. a library API) rather than invariants in the language or abstract machine.

Now, there are systems in which the abstract machine itself _does_ provide the necessary invariants for the paper's framework to work, for example CHERI. That said, most systems today use conventional architectures and unsafe languages and so must strive for the best locality/noninterference they can by using libraries, abstraction, and safe coding patterns to enforce invariants. In the future, it would be interesting to see the locality/noninterference argument extended with some sort of "composition of abstract machines" argument that allows encapsulation and APIs to rigorously restrict interference even in the face of internal reachability.

## Safe languages use invariants to provide memory safety, but these invariants do not define memory safety

Many definitions of memory safety talk about concepts like type safety, initialization safety, concurrency safety, etc. These are all necessary areas to consider when a language is carving out a subspace of the total space of possible program executions that is guaranteed to be free of memory unsafety. However, a program could in principle respect memory safety while violating any or all of these properties.

Probably the most controversial of these to exclude from a definition of memory safety is type safety, for two reasons. First, many common definitions of memory safety assume type safety, and it's an integral piece of how many security researchers view memory corruption. Second, modern languages often rely heavily on type safety as an underlying invariant that allows them to construct guarantees around memory safety.

However, I don't see anything about accessing memory through a pointer with the "wrong" type that fundamentally forbids one from building guarantees that ensure handle/object integrity, local reasoning about state, and noninterference. Type safety is simply a very nice property to lean on when building such guarantees in a language. Similar arguments could be made about other properties common to safe languages, including initialization and concurrency safety.

In general, I tend to distinguish the concepts of "memory safety" and "memory safety invariant". "Memory safety" is the property of not exhibiting memory unsafety. A "memory safety invariant" is an invariant relied on by the program or language to guarantee memory safety. That is, if all memory safety invariants are upheld, then memory safety is guaranteed. But violating a memory safety invariant does not mean that memory unsafety has occurred, just that it's no longer guaranteed not to occur.

Programs and languages use memory safety invariants to constrain the set of all possible executions of a program to a (strict) subspace of all memory-safe executions. But these invariants are stronger than memory safety. I honestly don't know what defines the perimeter of the space of _all_ memory safety, the exact boundary between memory safe and memory unsafe.

---

Thanks to Thomas Dullien, Jann Horn, and others.
