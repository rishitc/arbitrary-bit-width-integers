# Arbitrary Bit Width Integers in Carbon

## Keywords

Arbitrary bit width integers, custom-width integers and exact width integers.

## Feature Description

A language feature that lets users define integer types that behave similar to integers that occupy the user-specified number of bits.

**Example:** `i13` denotes an integer type which behaves similar to an integer which occupies 13 bits.

**Note:** I've used the term **_"similar to"_** and not _**"exactly"**_ primarily because although various interpretations of this feature agree upon its usage in a program, the actual implementation (e.g. in-memory representation) of this feature seems to be largely dependent on the underlying hardware and supporting infrastructure such as the compiler stack etc.

## Feature Pitch

Suppose you are working on a task that requires you to use values that fit in the integer range defined by 24 bits. In a programming language such as C++ or Java, the common practice would be to use an `int` type, which generally occupies 32 bits in C++ and is guaranteed to occupy 32 bits in Java.

With this feature, a developer could clearly specify in the code the integer width they require and enforce that statically at compile time, as well.

## Status Quo of Carbon

Currently, Carbon supports integers of bit widths that are a multiple of 8, e.g., `i8`, `i16`, `i24`, `i48`, `i64`, `i128`, and `i256` etc.

## Motivation

In most codebases, the usual 8-, 16-, 32-, and 64-bit width integer types provide satisfactory expressiveness for common integer uses. However, there are use cases for integer types with a specific bit-width in application domains, such as:

1. Using 256-bit integer values in various cryptographic symmetric cyphers like AES when calculating SHA-256 hashes

2. Representing a 24-bit colour space.

3. Describing the layout of a network or serial protocol. Example: The `header length` field of TCP protocol has a size of 4 bits.

4. In the case of FPGA hardware, using normal integer types for small value ranges where the full bit-width isn't used is extremely wasteful and creates severe performance/space concerns. At the other extreme, FPGA's can support really wide integers, essentially providing arbitrary precision, and existing FPGA applications make use of really large integers, for example, up to 2031 bits. [\[ref\]](https://blog.llvm.org/2020/04/the-new-clang-extint-feature-provides.html)

5. 4 and 12 bit would be nice for DNA/HDR images etc. [\[ref\]](https://github.com/JuliaLang/julia/issues/45486#issuecomment-1435833966) [\[ref-2\]](https://en.wikipedia.org/wiki/High-dynamic-range_television#:~:text=A%20bit%20depth%20of%2010%20or%2012%20bits%20is%20used,2100.)

6. A lot of embedded displays only have a color depths of 16 bits in which case the representation is usually something like (RGB565) [\[ref\]](https://github.com/JuliaLang/julia/issues/45486#issuecomment-1435920379) [\[RGB565\]](https://rgbcolorpicker.com/565) [\[Range limited types\]](https://therealprof.github.io/blog/roadmap-2021-arbitrary-size-primitives/)

7. If we use types that are a multiple of 8 bits, then in such cases, to ensure that the values provided are in the valid range, we need to add extra checks such as:  
i. Using a clamping technique to limit the values in the valid range  
ii. Use assertions to ensure that the values provided are in the valid range, if not, then raise an error.  
iii. Adding extra checks is cumbersome and error-prone. It would be more convenient if we could do something like [rust]  
iv. Additionally, the compiler can do a much better job at compile time to ensure that incorrect uses of such types will be noticed at compile time instead of panicking at runtime, for example, in the case of named constants. [\[ref\]](https://therealprof.github.io/blog/roadmap-2021-arbitrary-size-primitives/)

8. Camera hardware that uses 12-bit or 14-bit analog-to-digital conversion and then transfers those packed bits into a raw binary file. [\[ref\]](https://github.com/JuliaLang/julia/issues/45486#issuecomment-1436034808) [\[ref\]](https://cinematography.com/index.php?/forums/topic/66962-does-a-14-bit-adc-actual-produce-14-bit-dynamic-range-images/)

9. An integer type with an explicit bit-width allows programmers to express their intent in code more clearly without portability concerns. Today, the programmer must pick an integer datatype of the next larger size and manually perform mask and shifting operations.
However, this is error-prone because integer widths vary from platform to platform, and exact-width types from `stdint.h` is optional, it is not always obvious from code inspection when a mask or a shift operation is incorrect, implicit conversion can lead to surprising behaviour, and the resulting code is harder for static analysers and optimisers to reason about. By allowing the programmer to express the specific bit-width they need for their problem domain directly from the type system, the programmer can write less code that is more likely to be correct and easier for tooling and humans to reason about. [\[ref\]](https://www.open-std.org/jtc1/sc22/wg14/www/docs/n2763.pdf)  
 i. Regarding "portability", it would be on the compiler writers to ensure a consistent interface to the user, though the underlying implementation may change due to different target architectures (e.g. register sizes, assembly instructions available, etc.).

10. CPUs aren't the only backends that can benefit from these arbitrary integers, and those other backends wouldn't want extra padding/alignment added to values. For instance, newer GPUs are starting to include Int4 support in some of their cores, so they want native 4-bit integers without padding to 8-bits (@maleadt might be able to clarify how that actually works, but I would imagine it might want an LLVM `i4` as the type for those signals). Also, my original use case for arbitrary integer sizes was actually FPGA HLS backends, where when I say I want a 5-bit integer, it means I really only want 5 signal lines, so padding it to 8 lines (by doing an 8-bit integer) negates any performance/area/power savings of my design choice. [\[ref\]](https://github.com/JuliaLang/julia/issues/45486#issuecomment-1436169851).

11. As an embedded developer, I want to be able to safely represent memory-mapped registers as bitfields [\[ref\]](https://therealprof.github.io/blog/roadmap-2021-arbitrary-size-primitives/).

12. As a system developer, I want to be able to use automatically range limited types to prevent incorrect use and eliminate error checking [\[ref\]](https://therealprof.github.io/blog/roadmap-2021-arbitrary-size-primitives/)

13. An array of arbitrary bit-width integers can be used to create bitmaps, for example: [\[ref\]](https://therealprof.github.io/blog/roadmap-2021-arbitrary-size-primitives/)
```rust
// size of::<[bool; 10]>() == 10
let arr: [bool; 10] = [false; 10];

/// size of::<[u1; 10]>() == 2
let arr : [u1: 10] = [0; 10];
```

14. Using exact bit width integers in structs can automatically give us the feature of bit-fields like in C/C++.

Given points 13. and 14. we observe that arbitrary bit width integers come across as a more fundamental feature of the language on which other features can be derived, i.e., by adding support for arbitrary bit width integers to the language, we will automatically add support for bit fields and bitmaps without much extra effort.

### Why not use Larger Types + Bit Masks/Shifts?

Today, the programmer must pick an integer datatype of the next larger size and manually perform mask and shifting operations to get the desired behaviour.

However, it is not always obvious from code inspection when a mask or a shift operation is incorrect; implicit conversion can lead to surprising behaviour, and the resulting code is harder for static analysers and optimisers to reason about.
By allowing the programmer to express the specific bit-width they need for their problem domain directly from the type system, the programmer can write less code that is more likely to be correct and easier for tooling and humans to reason about.

Though this approach works, it does not express the exact domain constraints in the code, reducing code readability/expressivity while also occupying more memory and potentially being a cause for future bugs where values outside the range are generated by the code, but the programming language does not error any overflow.

This feature would allow programmers to specify and use integers with the bit width they desire, allowing for more precise control over the range and space occupied by the type.

## Implementation Status in Other Programming Languages/Compilers

### Zig

This feature has been implemented and is available in the language. [\[ref\]](https://ziglang.org/documentation/master/#Primitive-Types)

Provided below is a sample Zig program leveraging this feature:

```zig
const std = @import("std");

pub fn main() !void {
    var a: u4 = 5;
    var b: u4 = 7;

    var result: u4 = addNumbers(a, b);

    std.debug.print("The result is {}\n", .{result});
}

fn addNumbers(a: u4, b: u4) u4 {
    return a + b;
}
```

Note that the below program will panic at runtime due to an integer overflow:

```zig
const std = @import("std");

pub fn main() !void {
    var a: u4 = 9;
    var b: u4 = 10;

    var result: u4 = addNumbers(a, b);

    std.debug.print("The result is {}\n", .{result});
}

fn addNumbers(a: u4, b: u4) u4 {
    return a + b;
}
```

LLVM currently has support for arbitrary bit-width integers. This feature is leveraged by Zig to expose this feature in the language.

### Rust

This feature appears to have been discussed at length in the Rust internal forums as a pre-RFC [\[ref\]](https://internals.rust-lang.org/t/pre-rfc-arbitrary-bit-width-integers/15603) and a related discussion [\[What’s a “zero-bit integer”?\]](https://internals.rust-lang.org/t/whats-a-zero-bit-integer/15650). However, no work seems to be ongoing at the moment to get this feature into the language.

There was an introduction to the motivation, challenges and implementation ideas of the feature discussed in the blog [\[here\]](https://therealprof.github.io/blog/roadmap-2021-arbitrary-size-primitives/). The points related to motivation have been covered in the motivation section. Bit-fields were also covered in the motivation section; however, as @Josh11b mentioned on Discord, there is already a planned proposal for Bitfields for Carbon.

The implementation challenges raised for this feature are as follows:
1. Cannot get address easily:  
i. We cannot (easily) calculate and use a pointer to such a type, so taking a pointer would have to be forbidden.

2. Complier complexity:  
  i. The compiler might (or might not) be bogged down if we add dozens or even more than a hundred new types (depending on how far we want to take this), so we might potentially have to not enable those types by default and require explicit use statements instead, which should be fine.

3. Using a bitfield for memory-mapped-input-output (MMIO) might need some additional rules or some easier-to-find documentation than what's currently hidden in the nomicon [\[ref\]](https://doc.rust-lang.org/nomicon/other-reprs.html).

The implementation ideas are covered below:

1. Allow `#[repr(T)]` on structs if and only if the sizes match.

2. Add `ux` and `ix` types to the primitives.

3. Properly reflect them internally in the compiler to eliminate unnecessary bound checks.

4. It might be possible to hook those types directly into LLVM to use by code generation backends.

https://internals.rust-lang.org/t/pre-rfc-arbitrary-bit-width-integers/15603
rust-lang/rfcs#2581

**TODO:** Takeaways from the pre-RFC discussion [\[ref\]](https://internals.rust-lang.org/t/pre-rfc-arbitrary-bit-width-integers/15603) and a related discussion [\[What’s a “zero-bit integer”?\]](https://internals.rust-lang.org/t/whats-a-zero-bit-integer/15650).

**TODO:** Investigate further if this feature is being worked on or discussed in other forums. Probably check out Rust's Zulip community.

### C

This feature has been accepted into the C23 standard. [\[StackOverflow Link: Look at the comment underneath the accepted answer\]](https://stackoverflow.com/questions/73079098/how-to-make-dynamic-bit-width-integers-in-c) [\[Clang 18.0.Ogit documentation\]](https://clang.llvm.org/docs/LanguageExtensions.html#extended-integer-types)

### LLVM/Clang

https://llvm.org/docs/LangRef.html#integer-type
https://reviews.llvm.org/rG5f0903e9bec97e67bf34d887bcbe9d05790de934
https://reviews.llvm.org/rG6c75ab5f66b4
https://discourse.llvm.org/t/rfc-add-support-for-division-of-large-bitint-builtins-selectiondag-globalisel-clang/60329?u=programmerjake

Quoting from the Clang 18.0.Ogit documentation [\[ref\]](https://clang.llvm.org/docs/LanguageExtensions.html#extended-integer-types):
> Clang supports the C23 _BitInt(N) feature as an extension in older C modes and in C++. This type was previously implemented in Clang with the same semantics, but spelled _ExtInt(N). This spelling has been deprecated in favor of the standard type.
>
> Note: the ABI for _BitInt(N) is still in the process of being stabilized, so this type should not yet be used in interfaces that require ABI stability.

In LLVM, for types such as `i100`, it allocates a block of 100 bits on the stack (maybe with some alignment padding). Doing operations with such types is limited [\[ref\]](https://stackoverflow.com/questions/22255634/llvm-arbitrary-precision-integer) to whatever the CPU provides from its instruction set.
  The reference also quoted an interesting way to get the assembly output from LLVM bytecode for a specified architecture, using the LLVM static compiler utility `llc` to test out LLVM code and see what it compiles to. [\[ref\]](https://releases.llvm.org/1.2/docs/CommandGuide/llc.html)

### Julia

They were mainly tracking the feature in the C standard (specifically proposal N2709) before taking a decision on whether to move forward with the feature. In the discussion, it was later mentioned that this feature was accepted as [N2763: Adding a Fundamental Type for N-bit integers](https://www.open-std.org/jtc1/sc22/wg14/www/docs/n2763.pdf).

* The last round of `_BitInt(N)` fixes were made in [N3035](https://www.open-std.org/JTC1/SC22/WG14/www/docs/n3035.pdf).

* Refer to [N3088](https://www.open-std.org/JTC1/SC22/WG14/www/docs/n3088.pdf) which is the latest C23 draft and includes the latest spec for `_BitInt(N)`.

Even the Project Editor for C @ThePhD chimed in to encourage the feature implementation. [\[ref\]](https://github.com/JuliaLang/julia/issues/45486#issuecomment-1435772236)

Their initial implementation focuses on integers with a size of multiplies of 8 bits.
Some use case that require integer sizes that are not a multiple of 8 bits are 4 and 12 bits would be somewhat nice for DNA/HDR images [\[ref-1\]](https://github.com/JuliaLang/julia/issues/45486#issuecomment-1435833966)[\[ref-2\]](https://en.wikipedia.org/wiki/High-dynamic-range_television#:~:text=A%20bit%20depth%20of%2010%20or%2012%20bits%20is%20used,2100.) however they noted that rounding to the nearest byte isn't exactly unreasonable.

**Author's note:** It does put extra bit manipulation work on the programmer, which could have at least been hidden by the compiler underneath an integer of size 4 bits or 12 bits. This sentiment is echoed by another user later: [\[ref\]](https://github.com/JuliaLang/julia/issues/45486#issuecomment-1435920379)
> It's also worth considering older color formats as well, which may e.g. pack 5 bits per R, G, and B.
>
> Obviously, those pack into 16 bits nicely, possibly using extra bit for a mask, and so it's not that hard to manually unpack, do the math, mask, repack, etc. etc. But allowing people to not have to write such code, reimplementing it over and over, is what these types are for.

It was also noted that the compiler is free to add padding and will, therefore likely round up to bytes for performance. [\[ref\]](https://github.com/JuliaLang/julia/issues/45486#issuecomment-1436071290). However, if the hardware has support for the specified bit-width integer, then compiler a chose that as well. [\[ref\]](https://github.com/JuliaLang/julia/issues/45486#issuecomment-1436143217)

Another point raised was regarding the support for arrays of arbitrary bit width integers. For this the case of Zig was brought up where there are normal arrays have a memory layout where each element is a multiple of a byte. Via Godbolt, we can see an array of i9 of six elements takes up 12 bytes: [\[Godbolt link\]](https://godbolt.org/#g:!((g:!((g:!((h:codeEditor,i:(filename:'1',fontScale:14,fontUsePx:'0',j:1,lang:zig,selection:(endColumn:22,endLineNumber:3,positionColumn:22,positionLineNumber:3,selectionStartColumn:22,selectionStartLineNumber:3,startColumn:22,startLineNumber:3),source:'//+Type+your+code+here,+or+load+an+example.%0Aexport+fn+testArraySize()+i64+%7B%0A++++const+arr+%3D+%5B_%5Di9%7B1,+2,+3,+4,+5,+6%7D%3B%0A++++return+@sizeOf(@TypeOf(arr))%3B%0A%7D%0A'),l:'5',n:'0',o:'Zig+source+%231',t:'0')),k:33.333333333333336,l:'4',n:'0',o:'',s:0,t:'0'),(g:!((h:compiler,i:(compiler:z0100,deviceViewOpen:'1',filters:(b:'0',binary:'1',binaryObject:'1',commentOnly:'0',demangle:'0',directives:'0',execute:'1',intel:'0',libraryCode:'0',trim:'1'),flagsViewOpen:'1',fontScale:14,fontUsePx:'0',j:1,lang:zig,libs:!(),options:'',selection:(endColumn:1,endLineNumber:1,positionColumn:1,positionLineNumber:1,selectionStartColumn:1,selectionStartLineNumber:1,startColumn:1,startLineNumber:1),source:1),l:'5',n:'0',o:'+zig+0.10.0+(Editor+%231)',t:'0')),k:33.333333333333336,l:'4',n:'0',o:'',s:0,t:'0'),(g:!((h:output,i:(compilerName:'zig+0.10.0',editorid:1,fontScale:14,fontUsePx:'0',j:1,wrap:'1'),l:'5',n:'0',o:'Output+of+zig+0.10.0+(Compiler+%231)',t:'0')),k:33.33333333333333,l:'4',n:'0',o:'',s:0,t:'0')),l:'2',n:'0',o:'',t:'0')),version:4)
Zig also has a `PackedIntArray` in its standard library. The documentation of the packed configuration is: [here](https://ziglang.org/documentation/master/std/#A;std:PackedIntArray)

They are pursuing a similar pattern. Vector{UInt12} should allocate 2 bytes per element. Bit packing should probably be left to packages that could utilize specialized SIMD routines for packing and unpacking. [\[ref\]](https://github.com/JuliaLang/julia/issues/45486#issuecomment-1436457630)

#### Points for the feature

1. For sizes larger than 64 bits, it might not have benefits over an efficient `bigint`.

2. LLVM backend ingests the non-standard integers and converts them to a form usable value in the backend (which may imply that byte alignment is done).

3. An interesting use case brought up (but not named) was not to have any alignment done before passing to the backend. The eventual goal was to try taking the LLVM IR and feeding it into the OneAPI or Vitis HLS toolchains and get the hardware out of it. So, for the target hardware, they didn't want padding because that padding would negate the benefit of using a smaller integer type in the first place. [\[ref\]](https://github.com/JuliaLang/julia/issues/45486#issuecomment-1140327025)

## Assembly level Support

In the blog post [\[ref\]](https://alic.dev/blog/custom-bitwidth), the author shares that the general way of extracting an oddly sized unaligned integer out of a dense array is using the shift and mask technique. They argue that the cost of this is minimal for the data compression and cache line efficiency you get as the shifting and masking operations are all going to be done using the CPU registers; so effectively, we have placed more data closer to the CPU while only minimally increasing the number of operations needed to extract the data.

Instead of a shift and mask approach, the blog mentions that we can use `BEXTR` on x86, which is part of the BMI1 extension (post-Haswell). The equivalent on ARMv6+ is `UBFX` and on PowerPC, we have `rlwinm`.

On x86, the `BEXTR` instruction appears to have varying performance and latency numbers across AMD and Intel; even in the worst case, it is marginally better than using `shift + and` instead. A common benefit across all use cases appears to be code size, as now you only need one instruction instead of two. AMD's implementations generally appear to be better than Intel's. Overall, the discussion highlighted `BEXTR` as a viable alternative to using `shift + and`.
A GNU GCC email thread covers this well [\[ref\]](https://gcc.gnu.org/bugzilla/show_bug.cgi?id=89063).

On ARM, the `UBFX` instruction takes 1 cycle. [\[ref\]](https://developer.arm.com/documentation/ddi0338/g/cycle-timings-and-interlock-behavior/bitfield-instructions).

## References

1. [N2709](https://www.open-std.org/jtc1/sc22/wg14/www/docs/n2709.pdf)

1.
```rust
// The types will do the type-checking for us
pub fn set_pixel_rgb565(&mut self, r: u5, g: u6, b: u5) {
    ...
}
```