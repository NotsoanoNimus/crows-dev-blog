---
tags:
  - c3
  - language-development
  - cryptography
  - tests
title: 2025-08-19 - Dreams of SHA-3
permalink: work/2025-08-19
publish: true
---
> [!info]- Article PGP Signature
>
> To verify this signature, use the public key on the [[Hello, World!|Home Page]] and follow the instructions linked there.
>
> ```
> -----BEGIN PGP SIGNATURE-----
> 
> iHUEABYKAB0WIQT2vFo+jLf+FWIQ2T8xrKfG/dldbgUCaKYxXgAKCRAxrKfG/dld
> brgnAQDzFKmq5bLzZGxLUN2g7b45VKXxnLVxJ07of2Nzk5ajAgD8CfQINZcsNPDT
> CJXItPGPHrZ1N81a5dJasVnQwx0L4AY=
> =25V/
> -----END PGP SIGNATURE-----
> ```
>

## Today's Topics: SHA-3 and Next-gen Hashing
#####  August 20th, 2025, ca. 14:00 -0400 (_written a day late_)

---
### Contributions
- Branch: [sha3-shake-keccak](https://github.com/NotsoanoNimus/c3c/commit/264289a918d0d15d337e800d83ed024083f4126c)
  - Starting from most recent on *20250819*:
  - [264289a918d0d15d337e800d83ed024083f4126c](https://github.com/NotsoanoNimus/c3c/commit/264289a918d0d15d337e800d83ed024083f4126c)
  - [34bb58ea126ccf3bfc2775b2c3cc53697d730c3f](https://github.com/NotsoanoNimus/c3c/commit/34bb58ea126ccf3bfc2775b2c3cc53697d730c3f)
  - [4f45b9a7bc975287d8857b54737f6281e198aca7](https://github.com/NotsoanoNimus/c3c/commit/4f45b9a7bc975287d8857b54737f6281e198aca7)
  - [220670d3a57022c45c6a864a51c3a7be29fe93c9](https://github.com/NotsoanoNimus/c3c/commit/220670d3a57022c45c6a864a51c3a7be29fe93c9)
  - [02eea3c9038c55b6bfc90ea032df6765d70af8fd](https://github.com/NotsoanoNimus/c3c/commit/02eea3c9038c55b6bfc90ea032df6765d70af8fd)

### Bringing SHA-3 Online (for C3)
Yeah, putting together a [Whirlpool addition](https://github.com/c3lang/c3c/pull/2273) to the C3 standard library was certainly a cool experience. So was [fixing SHA-256](https://github.com/c3lang/c3c/pull/2247). And so was [implementing SHA-512](https://github.com/c3lang/c3c/pull/2227).

But... ==I still wanted to do more==.

Then it occurred to me:
1. which languages have a ready-to-use [SHA-3 suite of functions](https://en.wikipedia.org/wiki/SHA-3)?
2. what even are those functions? (_I didn't know them by name at all really_)

To answer the first question -- it turns out -- [most languages](https://en.wikipedia.org/wiki/SHA-3#Implementations) have this already in some library/package or another.

Fewer languages, however, have this functionality built into a standard library shipped with the language itself. Odin, for example, has many SHA-3-family functions [within its core library](https://pkg.odin-lang.org/core/crypto/sha3/). For the sake of full transparency, this is what I'm striving to emulate with my changes.

For the second question, well, that has been valuable knowledge gained along the way.

---
##### Basics: the Specifications
You can find the suite of SHA-3 functions _formally_ defined by the [National Institute of Standards and Technology](https://nist.gov/) (NIST) in the publication [FIPS-202](https://csrc.nist.gov/pubs/fips/202/final).

When I think about United States government organizations and how bureaucratic they are, I'm **continually surprised** at how well-organized, detailed, and just plain _helpful_ NIST is when trying to learn anything about cryptography and cryptographic standards.

I won't bother to explain the SHA-3 algorithms or fundamentals in great detail for this article, but I will mention [Keccak](https://csrc.nist.gov/csrc/media/projects/hash-functions/documents/keccak-slides-at-nist.pdf), the algorithm which underlies everything. It's fascinating how one mathematical construct can be so versatile and be employed in so many different ways to achieve different ends!

You should definitely check out the [Keccak Algorithm Visualizer](https://visualizekeccak.com/theory) to learn more.

SHA-3 formalizes the application of a specific Keccak permutation function for different fixed-length message digest outputs (the usual `224`, `256`, `384`, and `512` we know and love). It also formalizes a brand-new cryptographic primitive: the extendable-output function (or `XOF`).

---
##### What in the World is an `XOF`?
An [eXtendable-Output Function (XOF)](https://en.wikipedia.org/wiki/Extendable-output_function) allows a cryptographic hash to be extended to a theoretically infinite length. This concept is something that's pretty foreign to people in the cryptography space, even if they're familiar with classical primitives (hashing, asymmetric encryption, stream ciphers, etc.).

Traditional cryptography teaches that a hash (or _digest_) is a fixed-length output based on the selected algorithm. For example, `SHA-512` always produces 512-bit results. You could trim this output -- at the cost of increasing collisions -- but you could never extend it. XOF algorithms ___can___ extend it.

> [!example] Using the Sponge Function for Extendable Outputs
> 
> I'm going to use ***pseudo-code***, but it's not far off from the C3 library tests.
> ```swift
> char[] at_once = xof_func::get_128(16, "some input value"); // 16 bytes
>   // at_once: 1234567890abcdef1234567890abcdef
> 
> char[16] piecewise;
> Xof128 stream;
> stream.init();
> stream.update("some input value");
> stream.squeeze(piecewise[0:7]); // piecewise[0:7] = 1234567890abcd
> stream.squeeze(piecewise[7:7]); // piecewise[7:7] = ef1234567890ab
> stream.squeeze(piecewise[14:2]); // piecewise[14:2] = cdef
> 
> assert(at_once[..] == piecewise[..]); // true statement
> ```
> The `squeeze` function is the by-code act of "wringing out" or "squeezing" the sponge function (Keccak) in order to produce a result.
> 
> Each time the sponge is squeezed, the provided output buffer/slice is filled up entirely. Both the amount of buffers that can be filled and their sizes have a *theoretically infinite* limit.
> 
> With any XOF, it doesn't matter if I `squeeze(32)` or `squeeze(8) 4x`, the same data will be produced from the same input conditions -- allowing XOFs to behave as extensible hash functions.
> 
> While some SHA-3-derived algorithms might give varying output results upon squeezing, this is a deterministic output value when using XOF functions specifically.  For example, the `KMAC` construct changes its output depending on the length of the output buffer given, but if a `squeeze` is used instead of the `final` method, the results are always deterministic -- exactly like this sample code.

SHA-3 employs `SHAKE128` and `SHAKE256` as the most primitive XOF functions in the suite (each with a different "security level"); again, relying on the Keccak permutation/sponge function to achieve its variable-length output capability.

Other SHA-3-family constructs are then built on top of `SHAKE`, including `cSHAKE` (a [domain-separated](https://en.wikipedia.org/wiki/Domain_separation) version of `SHAKE`), `TurboSHAKE`, `KangarooTwelve`, and others. You can find a Special Publication from NIST which describes some of these derived SHA-3 functions here: [NIST SP 800-185](https://csrc.nist.gov/pubs/sp/800/185/final).

> [!tip] cSHAKE Parameters and Domain Separation
> Focusing on `cSHAKE` for a moment: the domain-separation of that function is a key facet which enables its extension into other areas, like `Keccak Message Authentication Codes (KMAC)` (see _SP 800-185_).
> 
> `cSHAKE` is built atop `SHAKE` and as expected it comes in security strengths of `128` and `256`. However `cSHAKE` accepts two new parameters:
> ```swift
> cSHAKE128(X, L, N, S), where...
>     X = the main input string,
>     L = the requested XOF output length (in bits)
>     N = a "function-name" string, used and standardized by NIST
>     S = a customization bit-string to define sub-variants
> ```
> Introducing `S` and `N` adds domain-separation into the mix. This means that all outputs from `cSHAKE` are completely different when either of these inputs are mutated.
> 
> `N` is an input string standardized and defined by NIST to define the name of the sub-function of `cSHAKE` being used; e.g., `KMAC`, `TupleHash`, `ParallelHash`, etc. This is set to a blank value when using the normal `cSHAKE` function.
> 
> `S` is an optional _customization string_ input, which further delineates the output from `cSHAKE` into a different category. The way this is described is: think of a few different domains where `cSHAKE` might be used... If you're using it to hash email signatures, `S` can be `email signature`. If you're using it to hash network daemon traffic, you could use the process name for `S`. So on and so forth.
> 
> So long as any party(ies) expected to reconstruct your hashes are aware of which input parameters you used, you won't run into any problems generating the same deterministic outputs.

Anyway, that's probably enough prattling about these \[\[very neat\]\] functions for now.

---
##### TupleHash is the Coolest Thing...
Relevant to the work of 2025-08-19, and amidst all of the code renovation I completed, I want to emphasize and ==highlight== the newly-implemented **TupleHash**. I'm going to let NIST do the talking for me on this one...

In the aforementioned [NIST SP 800-185](https://csrc.nist.gov/pubs/sp/800/185/final) publication is where TupleHash is defined:

> TupleHash is a SHA-3-derived hash function with variable-length output that is designed to simply hash a tuple of input strings, any or all of which may be empty strings, in an unambiguous way. Such a tuple may consist of any number of strings, including zero[...].
> 
> TupleHash is designed to provide a generic, misuse-resistant way to combine a sequence of strings for hashing such that, for example, a TupleHash computed on the tuple `("abc", "d")` will produce a different hash value than a TupleHash computed on the tuple `("ab", "cd")`, even though all the remaining input parameters are kept the same, and the two resulting concatenated strings, without string encoding, are identical.
> 
> TupleHash supports two security strengths: 128 bits and 256 bits. Changing any input to the function, including the requested output length, will almost certainly change the final output.

Right, so in summary: with `TupleHash`, inserting input tuples in ***different orderings*** produces distinct, _unrelated_, deterministic outputs (which are extendable!).

```swift
test::ne( // ne -> not equal
   tuple_hash::xof_256(34, "mind", "bending", "hash", "tricks!"),
   tuple_hash::xof_256(34, "mindbending", "hashtricks!")
);   // returns true
```

These two have the exact same parameters. The only difference that is producing distinct hashes is the **order** in which the inputs are consumed by the hash function.

I mean -- come on! -- this is the absolute ==<u>coolest thing</u>== I've learned about since I familiarized myself with post-quantum SLH-DSA (SPHINCS).

![[mind_blown.gif]]

---
##### The Importance of Unit Tests
It is _essential_ to have [thorough coverage with unit tests](https://tsapps.nist.gov/publication/get_pdf.cfm?pub_id=924038) when implementing any cryptographic algorithm. If you don't cover all branches and you make a *cryptic* (heh) mistake, you're bound for trouble later when the worst happens. Not only that, but mistakes in your implementation will inevitably let down others who depend upon your meticulous work.

Even today, it's hard to find good online resources, vectors, calculators, etc. for the SHA-3 family of functions. Some of the functions seem *very* obscure. At the time of writing, not even [Cyber Chef](https://gchq.github.io/CyberChef/) has most of the good stuff. This is despite the fact that NIST has had these formalized for over a decade now.

Aside from setting up a harness to employ a flavor of the NIST [Automated Cryptographic Validation Protocol](https://github.com/usnistgov/ACVP), you should make full use of NIST's already-given test vectors. For example, here are some [ready-made `SHAKE128` test projections](https://github.com/usnistgov/ACVP-Server/blob/master/gen-val/json-files/SHAKE-128-1.0/internalProjection.json) that you can thumb through. While the JSON file does a well-enough job of describing each field of the test projections, it's always worth checking out the [related standards](https://pages.nist.gov/ACVP/draft-celi-acvp-sha3.html) for explicit instructions.

Anyway, beyond my own manual inclusion of some of the NIST vectors, I've used another method to generate tests. With the help of a [companion python script](https://github.com/NotsoanoNimus/c3c/blob/264289a918d0d15d337e800d83ed024083f4126c/test/unit/stdlib/hash/keccak/generate.py), I was able to reliably use the python's [pycryptodome](https://pypi.org/project/pycryptodome/) package to generate some simple non-standard test vectors for each SHA-3-family function along the way.

This has been a brief foray into *test-driven development*, where I was writing tests to stress the algorithms and create edge-cases, before even finishing implementation of the algorithms.

If you follow the link to the script above, you'll notice that the constructed dictionary printed from the `main()` function is the result of whichever handpicked families of functions I wanted to get information for at the time.

I removed the  comments and spacing for this copy-paste:
```python
def main():
	res = {
		'KECCAK': get_digests([224, 256, 384, 512], keccak_digest),
		'SHA3': get_digests([224, 256, 384, 512], sha3_digest),
		'SHAKE': get_xofs([128, 256], [16, 256], [''], shake_xof),
		'TURBO-SHAKE': get_xofs([128, 256], [16, 256], [''], turboshake_xof),
		'CSHAKE': get_xofs([128, 256], [64], sample_custom_strs, cshake_xof),
		'KMAC': get_kmacs([128, 256], [32, 64, 128], sample_kmac_keys, sample_custom_strs, kmac),
	}
	pprint.pprint(res)
```
This script pretty-printed the dictionary of test vectors for me to copy-paste into my C3 unit tests while writing them. This was simply a cheap way to get this information from an external source without needing to manually select and walk through individual NIST test vectors for **every. single. test.** (and there are many of them).

Of course, once the tests are all completed and functional, I will prune this python script from the branch. No sense in leaving it around; this article both documents here how it was all done and links back to it anyway.

---
### Wrapping Up
On [[2025-08-18 - @reduce and @filter|Monday]] I had mentioned that today would likely be a "lighter day" -- _not so_! I changed a **lot** of code around and introduced some more awesome constructs into the C3 language, as documented here. :)

I plan to continue ==refining and renovating== this code until it's production- or stdlib-ready. My plan is to immediately propose this (well-tested) library for review by the C3 team for stdlib inclusion once it's thoroughly completed.

As it's already Wednesday 2025-08-20 when I'm catching up with this article, I can confidently say that more exciting things have happened in my branch since this log! You can read about those in the [[2025-08-20 - XOF XOF Crypto Rocks XOF XOF|next article]].