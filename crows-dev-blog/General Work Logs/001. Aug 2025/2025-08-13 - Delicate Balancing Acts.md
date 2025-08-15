---
tags:
  - c3
  - uefi
  - language-development
title: 2025-08-13 - Delicate Balancing Acts
permalink: work/2025-08-13
publish: true
---
#####  August 13th, 2025, ca. 19:00 -0400
## Today's Topics: [C3](https://github.com/c3lang/c3c) changes aplenty
> [!info]- Article PGP Signature
>
> To verify this signature, use the public key on the [[Hello, World!|Home Page]] and follow the instructions linked there.
>
> ```
> -----BEGIN PGP SIGNATURE-----
> 
> iHUEABYKAB0WIQT2vFo+jLf+FWIQ2T8xrKfG/dldbgUCaJ0kXgAKCRAxrKfG/dld
> biVoAQD98FH8a8qHGLLcbC2cLN6JW4w5Yzfr92gzyiRRdNswAAEA4YdmghyhP3Y3
> y63DiwQ5lZrtnn7FNx8Ub4p/mOUHzQw=
> =ecGx
> -----END PGP SIGNATURE-----
> ```
>

### Today's Contributions
- PR: polished and finally marked for review [c3lang/c3c#2392](https://github.com/c3lang/c3c/pull/2392)
- PR: [c3lang/c3c#2399](https://github.com/c3lang/c3c/pull/2399)
- PR: [c3lang/c3c#2400](https://github.com/c3lang/c3c/pull/2400)
- PR: [c3lang/c3c#2401](https://github.com/c3lang/c3c/pull/2401)
- Issue: [c3lang/c3c#2402](https://github.com/c3lang/c3c/issues/2402)

### UEFI: Progress, Progress
Yesterday's fiasco with the `@weak` attribute not working on Win32 (PE32+) binary formats left a bad taste in my mouth for the future of UEFI development with C3.

I logged off my workstation for the night a bit disappointed that this feature didn't seem reachable. This feeling wasn't helped when reading [Nim's struggle](https://forum.nim-lang.org/t/11228) with the same concept in a last-ditch deep search for a solution.

But today, upon revisiting the issue (and after some discussions in the Discord), I think I came up with a good compromise. First, I'll [add in](https://github.com/c3lang/c3c/pull/2399) some new `FREESTANDING` boolean constants into the runtime environment variables:

###### lib/std/core/env.c3
```cpp
const bool FREESTANDING_PE32 = NO_LIBC && OS_TYPE == WIN32;
const bool FREESTANDING_MACHO = NO_LIBC && OS_TYPE == MACOS;
const bool FREESTANDING_ELF = NO_LIBC && !env::FREESTANDING_PE32 && !env::FREESTANDING_MACHO && !env::WASM_NOLIBC;
const bool FREESTANDING = env::FREESTANDING_PE32 || env::FREESTANDING_MACHO || env::FREESTANDING_ELF || env::WASM_NOLIBC;
```

After doing this, I'll create multiple _conditional declarations_ of the `@weak`-linked `libc` module functions. This is a fancy way of saying "**if PE32, then search `extern`; else, use weak symbols**":

###### lib/std/libc/libc.c3
```cpp
module libc @if(env::NO_LIBC);
// modified original (notice the @if)
fn void* malloc(usz size) @weak @extern("malloc") @nostrip
	@if(!env::FREESTANDING_PE32) {...}
fn void* calloc(usz count, usz size) @weak @extern("calloc") @nostrip
	@if(!env::FREESTANDING_PE32) {...}
fn void* free(void*) @weak @extern("free")
	@if(!env::FREESTANDING_PE32) {...}
fn void* realloc(void* ptr, usz size) @weak @extern("realloc") @nostrip
	@if(!env::FREESTANDING_PE32) {...}
// additional conditional declarations for freestanding PE32
extern fn void* malloc(usz) @nostrip @if(env::FREESTANDING_PE32);
extern fn void* calloc(usz, usz) @nostrip @if(env::FREESTANDING_PE32);
extern fn void* free(void*) @nostrip @if(env::FREESTANDING_PE32);
extern fn void* realloc(void*, usz) @nostrip @if(env::FREESTANDING_PE32);
```

Now the behavior stays the same, until someone (like myself) comes along to compile a Windows-style executable format without linking `libc`. In such a case, the linker expects these symbols to exist _somewhere else_ during that phase of the process, so we evade the symbol collision.

With that out of the way, the next step of the process was getting a working `--panicfn` panic function exported from my `uefi::` module, which was promptly completed.

This revealed the next, blocking phase of the project: the `@dynamic` interface implementations are not properly being registered during the initialization of the runtime. Great.

> [!faq]
> It turns out, after chatting with C3's head developer, interface dynamic calls are--yep--_dynamically_ entered by an `@init` shim that runs at the program entrypoint. Well since I'm using `--no-entry` and defining my own (with my own runtime), dynamic interface function pointers are not being linked together properly. **Great**.

Thus this wing of the project will have to wait until I can work on this in more detail with people _much_ smarter than I am.

### Stomping on Benchmark & Git Annoyances
I amended the `.gitignore` finally so I can officially stop being bothered by the `benchmarkrun` executable constantly showing up in my index. I've clicked it too many times!

I also fixed an issue with benchmark printing causing some serious slowness when there were a significant number of iterations on each bench test. I only noticed this because I went to run my `non_crypto_shootout.c3` benchmark file as a test to get `benchmarkrun` to reappear in my fresh index.

### Compile-time and Runtime Ranging
I've recently added a [pending feature](https://github.com/c3lang/c3c/pull/2392) to the C3 standard library that can make dynamically creating a sequence of integers in an array much smoother, and more versatile than `math::iota`. This is an augmented and _strongly-typed_ [python `range` analog](https://docs.python.org/3/library/stdtypes.html#typesseq-range) for C3.

Today I spent some time adding type selection for the returned values and bolstered the unit tests a bit, before finally marking the PR _ready to review_.

> [!example] Range in Action
> ```cpp
> foreach (i : array::trange(200, step: 25, inclusive: true)) z += i;
> assert(z == 0 + 25 + 50 + 75 + 100 + 125 + 150 + 175 + 200);
> ```
> From `0`, walk up to the value `200` (inclusive, meaning the `200` is included in the end result), at a step of `25` per array element.
> 
> As the `assert` indirectly shows, this returns a ___runtime___ array of:
> `{0, 25, 75, 100, 125, 150, 175, 200}`
> 
> ```cpp
> assert({126, 127, -128, -127, -126} == @range(126, 130, $inclusive: true, $OfType: ichar));   // test signed overflows
> ```
> This creates a ___compile-time___ array of integers, starting from the _raw_ value `126`, up to and including `130`. However, `$OfType` forces the result to be boxed into an array of `ichar` (signed-byte) values having a range from `-128` to `127`.
> 
> Thus, when the raw value is `128`, exceeding the range limit of `ichar`, the cast causes an overflow, and the stored value in the array wraps back around to the bottom of the `ichar` range (so `-128`).
> 
> This is just testing that the behavior is working as expected when this edge case is encountered.

### In Closing...
Today was a whirlwind of different concepts and ideas, all centering around my desire to get something moving for a freestanding C3 environment, where I can hopefully spread my wings and develop CrOwS from a high-level at the beginning.

I must pause until the weekend when C3's creator can help me get back on track. Until then, I'll probably be revisiting ==Keccak== and ==SHA-3== in the standard library.

Ciao!