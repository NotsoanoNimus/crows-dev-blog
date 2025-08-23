---
tags:
  - TAGS
title: 2025-08-21 - TITLE
permalink: work/2025-08-22
publish: false
---
> [!info]- Article PGP Signature
>
> To verify this signature, use the public key on the [[Hello, World!|Home Page]] and follow the instructions linked there.
>
> ```
> -----BEGIN PGP SIGNATURE-----
> -----END PGP SIGNATURE-----
> ```
>

## Today's Topics: ParallelHash & String `Trim` Speeds
#####  August 22nd, 2025, ca. 14:00 -0400 (_written a day late_)

---
### Contributions
- Iterating on: [c3lang#2407](https://github.com/c3lang/c3c/pull/2407)
- Branch: [sha3-shake-keccak](https://github.com/c3lang/c3c/commit/8295dc6974ff8b00a3bccf17d505224c3ba85375)
  - Not going to list today's commits; the list is in the branch.

### Hashing in Parallel with SHA-3
The

---
### String Trimming: Search vs. ASCII Bitmaps
About a week ago, I submitted [this Pull Request](https://github.com/c3lang/c3c/pull/2407) which simply added the [vertical tab](https://symbl.cc/en/000B/) `\v` and the [form feed](https://en.wikipedia.org/wiki/Page_break#Form_feed) `\f` ASCII escape characters into the default set of the `String.trim` methods, within the core `string` module.

What's funny about this PR is that a 6-character change became an _entire_ benchmarking conversation and exploration of different optimizations for `trim`. We'll get to that in a second.

I started by defending the need for these two characters to be added; since, although they are not as common nowadays, it is the "expected" behavior of a C/C++ analog to include these if those already do (see: [C++'s `std::isspace`](https://en.cppreference.com/w/cpp/string/byte/isspace.html) and [C's `isspace`](https://en.cppreference.com/w/c/string/byte/isspace)).

---
### Summary
Closing thoughts.