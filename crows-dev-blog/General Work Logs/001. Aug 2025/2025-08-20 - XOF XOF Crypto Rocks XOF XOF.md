---
tags:
  - c3
  - language-development
  - cryptography
title: 2025-08-20 - XOF XOF Crypto Rocks XOF XOF
permalink: work/2025-08-20
publish: true
---
> [!info]- Article PGP Signature
>
> To verify this signature, use the public key on the [[Hello, World!|Home Page]] and follow the instructions linked there.
>
> ```
> -----BEGIN PGP SIGNATURE-----
> 
> iHUEABYKAB0WIQT2vFo+jLf+FWIQ2T8xrKfG/dldbgUCaKaNtAAKCRAxrKfG/dld
> bsSMAQD2D8to6j3RGt3or/G3FU1l6/0tYe6vR4HRec9CN+FzowD/e6CI6yb/mhWz
> Ux/kUpffhRo+vuPoyuGlS3HHtwSq1A0=
> SSSSSS=XyMD
> -----END PGP SIGNATURE-----
> ```
>

## Today's Topics: SHA-3 and Gang
#####  August 20th, 2025, ca. 22:00 -0400

---
### Contributions
- PR: [c3lang#2424](https://github.com/c3lang/c3c/pull/2424)
- Branch: [sha3-shake-keccak](https://github.com/NotsoanoNimus/c3c/commit/264289a918d0d15d337e800d83ed024083f4126c)
  - Starting from most recent on *20250820*:
  - [875fe1c11e4bbeb43413bef4287fdccc7ecc4752](https://github.com/c3lang/c3c/commit/875fe1c11e4bbeb43413bef4287fdccc7ecc4752)
  - [0816c39e4697b891cdffee0f23772da35691ebf4](https://github.com/c3lang/c3c/commit/0816c39e4697b891cdffee0f23772da35691ebf4)
  - [7d10560ca41c337af5964fd4bcafcdbee8448036](https://github.com/c3lang/c3c/commit/7d10560ca41c337af5964fd4bcafcdbee8448036)
  - [20df0eb164ea181df182d3861123f7de33088ea2](https://github.com/c3lang/c3c/commit/20df0eb164ea181df182d3861123f7de33088ea2)

### The Innards of the XOF
Welcome back!

In [[2025-08-19 - Dreams of SHA-3|yesterday's article]], I gave the full rundown on SHA-3 and my role in its integration within the C3 standard library as a built-in suite of utilities - well, not "built-in" in the "C3" sense, but just something that ships with the compiler by default. In that posting, I covered _eXtendable-Output Functions_ (or "XOFs" for short) and what they do.

> [!tip] Refresh Your Memory
> Recall that a hash is a *fixed-length output* which cannot be extended and should not be shortened, while an XOF is like a "sponge" that can be repeatedly "squeezed" to extract the next bytes in a deterministic sequence of outputs.
> 
> The `final` (for hashes) and `squeeze` (for XOFs) functions do almost the same thing, except `final` is intended to be a one-time use that also *finalizes* the underlying SHA-3 function context.

Today I was able to accomplish getting a differentiation between a `hash` output and an `xof` output working at runtime. Before issues with `KMAC` testing arose yesterday, I honestly had no idea there was a distinction like this to be made, because I was slacking on reading the docs in detail.

I would like to briefly dig into some of the details of actually implementing that functionality, some of the challenges, and some of the work to do going forward.

---
##### Squeezing & Padding the Sponge
Way down deep at the `SHAKE` XOF level, I overrode the previously passed-through `squeeze` and `pad` functions.
- The former modification was to enable **true** XOF functionality for constructs built on top of `SHAKE` (so basically all of them). This is obviously essential.
- The latter modification was to __disallow__ external access to the `pad` function so that it couldn't be called accidentally.

You'll notice in my commits for the day that I also have a bit more happening based off of boolean switches for a compile-time `$is_xof` boolean in order to control XOF functionality while limiting code generation.

Anyway, to the second point, unintentional calls to `pad` were seriously wrecking all of my tests for almost two hours before I figured out what was going on... This is thanks to the lines reading:
```swift
if (!self.is_padded)
{
	self.k.pad();
	self.is_padded = true;
}
```

Even though the "guard" was there, direct `pad()` calls for `SHAKE` and children were still calling the Keccak `pad` function.

And, yes, the `pad` function probably isn't the only one exposed that really shouldn't be. I need to **tighten this API** and audit it before submitting anything.

> [!warning] Oof and Big Yikes, et al.
> Unfortunately, C3 currently exposes even `@local` function pointers on `inline` structs to their child structures (well beyond module scopes). I have *not* yet filed an Issue for this yet on GitHub. For example:
> 
> ```swift
> module aaa;
> import std::io;
> 
> struct AStruct
> {
> 	int something;
> }
> fn void AStruct.l_method(&self) @local => io::printn("local!");
> fn void AStruct.p_method(&self) => io::printn("public!");
> 
> 
> module main_test;
> import aaa;
> 
> struct BStruct
> {
> 	inline AStruct a;
> }
> 
> fn void main()
> {
> 	BStruct b;
> 	b.l_method();
> 	b.p_method();
> }
> ```
> 
> As you would ___not___ expect - though there's likely a great technical explanation - the first call won't error out; it will print `local!` without issue!
> 
> This is a problem for unwanted exposure of hidden `@local` and `@private` methods unintentionally. To be clear, I don't have a problem being able to access them -- it's the lack of _intent_ to want to use them, and thus using them by accident without explicitly importing them.

---
##### Maintaining Clean Dishware
I'm starting to understand why [Zig uses a state-machine](https://github.com/ziglang/zig/blob/cab6d752e8f6f6d3f538075840acc108238feda3/lib/std/crypto/keccak_p.zig#L205) for its underlying Keccak permutation function. However, I do not ultimately believe it is the right solution for C3's unique capabilities and situation.

It's tough to trust users who aren't "in the trenches" on this topic with an API that's easy to misuse - or worse, abuse. Though it could be argued that developers who do not know what SHA-3 is ==should probably avoid using it==.

If a user wants to use `KMAC` in the XOF mode, they must either call `kmac::xof_128/256` wholesale (*easy enough*), or use `KmacXOF128/256` for streaming data. When the latter is chosen, the `XOF` version of the structure should have a few guardrails that make it default to **squeezing** the output rather than **finalizing** it.

In this case, there are predefined method overrides for the `XOF`-style structs which redirect calls to `final` straight to `squeeze`. But how do we know when a user is finished with the Keccak state so it can be securely wiped? After all, an XOF - by definition - has a theoretically infinite output. So the _user_ will need to **know** that the state should be finalized. This generic flexibility and OOO-like behavior that I've built into the modules facilitates ease-of-use, but at a cost...

---
### An Aside for Meta-programming
With some of the recent changes I've been making, I realized I should dynamically detect when `assert` has been enabled or disabled during compile-time. This would help me to refuse certain SHA-3 mis-orderings during debug builds, but just `return` in Production builds (rather than proceeding _through_ the `assert` statements).

```swift
fn void MyStruct.method(&self)
{
	if (!other_thing.was_done)
	{
		$if env::COMPILE_OPT_LEVEL >= O2:
			unreachable("you cannot do thing 'x' before thing 'y'");
			return;   // panic; return if panic is optimized away
		$else
			$error "Other thing wasn't done before calling " +++ $$FUNC;
		$endif
	}
	// do normal stuff
```

Something like this is necessary because any misuse of `init`, `update`, `final`, `squeeze`, etc. in the wrong order will always be detected at runtime, given the branch is encountered during testing.

```swift
CShake128 c;
c.update("haha I break");   // <-- don't do this!
c.init(NONE, "my customization string");
c,final(result_buf[..]);
```

Again: guardrails for developers.

Thus, I've introduced [a Pull Request](https://github.com/c3lang/c3c/pull/2424) to fix the compile-time environment variables for `COMPILE_OPT_xyz` optimization details.

---
### Alright Alright Alright
Overall a comparatively chill day sharpening the blades on some of my recent work. I suspect tomorrow will be an exciting new dawn for the _ParallelHash_ construct, which you can read more about in [NIST SP 800-185](https://csrc.nist.gov/pubs/sp/800/185/final).

The fun thing about _ParallelHash_ is that I see ***no other implementations*** of it in the wild; no calculators -- nothing -- just the NIST test vectors from their ACVP server repository. This means, once successfully implemented, C3 might be one of the first to ship it as a standard component.

Looking forward to a lovely Thursday. Bye!