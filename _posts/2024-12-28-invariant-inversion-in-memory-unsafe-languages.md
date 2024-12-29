---
layout: post
title:  "\"Invariant inversion\" in memory-unsafe languages"
---

One way of seeing the difference between memory-safe and memory-unsafe languages is that in a memory-safe language, the invariants used to uphold memory safety only "lean on" invariants that are enforced entirely by the language, compiler, and runtime, while in a memory-unsafe language the invariants used to uphold memory safety can "lean on" programmer-created (and thus programmer-breakable) invariants. This latter case can lead to a weird situation that I call "invariant inversion", where code breaks a safe-looking logical invariant and ends up creating subtle memory unsafety issues.

## The invisible bug

Take the following puzzle, which I'm running on my M1 Mac:

```
// clang++ -std=c++17 -Wall -O2 puzzle.cpp -o puzzle
#include <cstdint>
#include <iostream>

constexpr uint16_t COUNT = 32;

struct DataSet
{
	uint16_t value[COUNT];
	bool filter[COUNT];
};

class Evaluator
{
	uint16_t keep[COUNT] = {};
	uint16_t discard = 0;
	uint16_t success = 0;
public:
	bool evaluate(const DataSet& data);
};

bool Evaluator::evaluate(const DataSet& data)
{
	uint16_t index = 0;
	for (uint16_t i = 0; i < COUNT; i++) {
		if (data.filter[i]) {
			keep[index++] = data.value[i];
		} else {
			discard = data.value[i];
		}
	}
	return success != 0;
}

int main()
{
	std::freopen(nullptr, "rb", stdin);
	struct DataSet data_set = {};
	std::fread(&data_set, sizeof(data_set), 1, stdin);
	Evaluator *e = new Evaluator();
	if (e->evaluate(data_set)) {
		std::cout << "success!" << std::endl;
	} else {
		std::cout << "fail!" << std::endl;
	}
	return 0;
}
```

If you haven't seen this bug class before, I highly recommend spending a moment to try and figure out how to make this program print success before reading the rest of this post.

Go on, spend a moment.

Okay, so you've read the code snippet. You probably noticed that `::evaluate()` only returns `true` when `success` has been set to a non-zero value, but `success` is not written anywhere in this code! There has to be memory corruption of some form for this to happen. And it must be spatial memory corruption since there's no temporal features of this code. So where is it?

The only spatially-variable write in the program is to `keep[index++]`, so this must be the location that writes out of bounds. But the loop runs exactly `COUNT` times, and `index` is at most incremented by 1 on each iteration, so `index` cannot grow above `COUNT`. And since `index` is at most `COUNT`, writing `keep[index]` is always in-bounds. So then how could this access possibly go out-of-bounds?

## Assumptions and optimizations

My system is an AArch64 system using Clang 16.0.0. Here's how this code compiles on [Godbolt](https://godbolt.org/z/Tdea4Y1rd) with similar settings:

```
Evaluator::evaluate(DataSet const&):
        mov     x8, xzr               ; i = 0
        mov     w9, wzr               ; index = 0
        add     x10, x0, #64          ; x10 = &this->discard
.LBB0_1:
        add     x11, x1, x8           ; x11 = &data.filter[i] - 0x40
        ldrh    w12, [x1, x8, lsl #1] ; w12 = data.value[i]
        add     x13, x0, w9, uxth #1  ; x13 = &this->keep[index]
        add     x8, x8, #1            ; i++
        ldrb    w11, [x11, #64]       ; w11 = data.filter[i]
        cmp     w11, #0               ; compare filter[i] to 0
        add     w9, w9, w11           ; index += data.filter[i]
        csel    x13, x10, x13, eq     ; x13 = eq ? &discard : &keep[index]
        cmp     x8, #32               ; compare i to 32
        strh    w12, [x13]            ; store value[i] into *x13
        b.ne    .LBB0_1               ; loop if i < 32
        ldrh    w8, [x0, #66]         ; w8 = this->success
        cmp     w8, #0                ; compare this->success to 0
        cset    w0, ne                ; ne ? 1 : 0
        ret
```

You may have noticed that the assembly increments `index` with `data.filter[i]` directly, even though `data.filter[i]` is a byte value. Essentially, the compiled code looks like this:

```
bool Evaluator::evaluate(const DataSet& data)
{
	uint16_t index = 0;
	for (uint16_t i = 0; i < COUNT; i++) {
	    uint8_t filter = *(uint8_t *)&data.filter[i];
	    uint16_t *slot = filter ? &keep[index] : &discard;
	    index += filter;
	    *slot = data.value[i];
	}
	return success != 0;
}
```

Written like this, it becomes much clearer what's going on: if `data.filter[i]`, seen as a byte rather than a `bool`, is neither `0` nor `1`, then the code above will be incorrect because `index` will be incremented by a value larger than 1. That's how `index` ends up going out-of-bounds.

Despite what it might look like, this isn't really a compiler bug. Clang is [allowed](https://github.com/llvm/llvm-project/issues/85018) to assume that a `bool`-typed variable, despite being 1 byte in size, will always hold a value that is either `0` or `1` (i.e., the top 7 bits of the byte will be 0). The C++ rules for integral promotions, integral conversions, and boolean conversions all behave correctly and uphold the required invariants for this optimization to work. So it's actually self-consistent so long as you don't "artificially" manufacture non-canonical boolean values.

The advantage of this kind of assumption is that it allows Clang to make optimizations like the one seen above. Codegen would probably be a bit worse if the compiler assumed that `bool`-typed memory slots could contain any bit pattern, and it would certainly be worse if the compiler explicitly narrowed `bool`-typed values to `0` or `1` every time they are read from memory.

## Spotting the bug

So, where exactly is the memory unsafety bug in the puzzle?

As we just discussed, the compiled code of `::evaluate()` is leaning on the rest of the system to maintain the invariant that any `bool`-typed memory slot only has the bit-pattern `0x00` or `0x01`. That is, `::evaluate()` can only be spatially unsafe if the `bool` invariant has already been violated elsewhere. And Clang does try to uphold that invariant so that compiler optimizations like the one above will be sound.

So where are the non-canonical `bool`s created? In this line:

```
std::fread(&data_set, sizeof(data_set), 1, stdin);
```

This line, which is essentially a `memcpy()`, is what sets the `bool`-typed fields of `DataSet` to non-canonical values, and is thus where the invariant relied upon by `::evaluate()` is broken. This is the undefined behavior on which the later memory corruption relies.

Unfortunately, undefined behavior in C++ codebases is common enough that this weird `bool` optimization can propagate it into actual memory unsafety [in the real world](https://www.nebula-graph.io/posts/troubleshooting-crash-clang-compiler-optimization).

## JIT compiler bugs

It may seem absurd that a line of code that fully respects spatial safety, a line that's basically just initializes a buffer, could be memory-unsafe. I certainly felt so when someone first showed me a bug like this. I suspect that many vulnerability researchers, especially those with a stronger background in C than C++, would probably find it hard to spot this bug from the source. Typically, in C codebases, you can attribute memory unsafety to the exact spot in the code that performs the invalid memory access, which in this case would be the assignment into `keep[index]`.

However, this bug might have a familiar feel for someone who's looked at exploiting optimizing JIT compilers like V8.

Optimizing JIT compilers want to make JITted JavaScript code fast. They do this by trying to check and enforce invariants that allow generating highly optimized code that would be unsound or unsafe if any of the invariants were ever violated. For example, V8 might perform a bounds-check of a value at the start of the function and then rely on the invariant that that value is within some expected range within the function body to eliminate subsequent bounds checks. If, due to some logic bug in the compiler, the value could be transformed during the function body to be outside the compiler's expected range, violating that invariant, then the optimized code may behave unsafely when run.

Just as with the `fread()` line above, it's not that the JIT compiler itself is doing memory corruption; the compiler could be written in Rust and the result would be the same. The `fread()` in the puzzle above is itself fully memory-safe with respect to _its own_ safety invariants.

However, the logical effects of the `fread()` violate invariants that other parts of the code rely on to guarantee memory safety. So the downstream effect is memory unsafety.

## Invariant inversion

This is what I mean by invariant inversion: unsafe languages (and, in the case of JIT, safe systems that interact with external forms of unsafety like RWX memory) can create chains of invariants leaning on other invariants such that you get an invariant at the bottom of that pile that looks totally innocuous, totally logical in nature, and yet at the top of that stack some memory safety property is relying on it. So breaking this logical-looking invariant, which you might assume would only produce functional bugs, actually results in a much deeper and more catastrophic effect on program integrity.

In the case of the puzzle above, the inverted invariant is that `bool`-typed variables must only contain the values `0` and `1`. I think of it as inverted because the canonical-`bool` invariant intuitively seems "higher level" than memory safety, and yet actually it is relied upon by the "lower level" memory safety property.

If you ever try to untangle the web of which invariants rely on which other invariants in a program by hand, you'll quickly find that it becomes unmanageable: the full invariant graph is simply too complicated to keep it all in your head at once.

I prefer this framing of looking at the graph of which invariants are being relied upon to uphold memory safety for a program because it sidesteps the somewhat distracting question of "where exactly is the memory unsafety?". I think that question comes from a very C-oriented view of the world, which modern systems are increasingly diverging from. Instead, I find it more helpful to ask the questions "where was the first violation of an invariant relied upon for memory safety?" and "where was the first invalid memory access?".
