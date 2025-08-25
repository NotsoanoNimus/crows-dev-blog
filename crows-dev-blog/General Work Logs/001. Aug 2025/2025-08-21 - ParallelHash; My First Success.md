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
#####  August 23rd, 2025, ca. 12:00 -0400 (_written two days late_)

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

What's funny about this PR is that a 6-character change became an _entire_ benchmarking conversation and exploration of different optimizations for `trim`. Honestly, I love that, it's the kind of optimization we \[_systems developers_\] should live for. I'll get to that in a second.

I started by defending the need for these two characters to be added; since, although they are not as common nowadays, it is the "expected" behavior of a C/C++ analog to include these if those already do (see: [C++'s `std::isspace`](https://en.cppreference.com/w/cpp/string/byte/isspace.html) and [C's `isspace`](https://en.cppreference.com/w/c/string/byte/isspace)). To which the C3 language designer, Christoffer Lernö, correctly pointed out that this will increase the time it takes for the input string to be searched. Ah, an **important detail** indeed.

Shortly thereafter, he also introduced the [`AsciiCharset` construct](https://github.com/NotsoanoNimus/c3c/blob/e4e499edd24862cc6e46bc09e0063625d3bb4438/lib/std/core/ascii.c3#L115) into the core standard library. This structure is simply a `typedef` to a `uint128` integer, and essentially acts as a bitmap for all 127 [7-bit ASCII](https://en.wikipedia.org/wiki/ASCII#Table_of_codes) characters. I thought this was _very_ clever as a branch-less solution to the speed problem.

```swift
typedef AsciiCharset = uint128;
macro AsciiCharset @create_set(String $string) @const //compile-time
{
  AsciiCharset $set;
  $foreach $c : $string: $set |= 1ULL << $c; $endforeach
  return $set;
}
fn AsciiCharset create_set(String string) // runtime
{
  AsciiCharset set;
  foreach (c : string) set |= (AsciiCharset)1ULL << c;
  return set;
}
macro bool AsciiCharset.contains(set, char c) // no branching!
  => !!(c < 128) & !!(set & (AsciiCharset)(1ULL << c));
const AsciiCharset WHITESPACE_SET = @create_set("\t\n\v\f\r ");
const AsciiCharset NUMBER_SET = @create_set("0123456789");
```

So cool: this is the first time I've encountered a bitmap used for this specific purpose. To visualize this more clearly, an  example helps. Imagine you have the search set `\x20\t\n\r\f\v` (`[space]` is `0x20`). To create a 128-bit integer -- or the number underlying the `AsciiCharset` filter -- out of this resolves to:

```swift
uint128 set =
  (1ULL << ' ') | (1ULL << '\t') | (1ULL << '\n')
  (1ULL << '\r') | (1ULL << '\f') | (1ULL << '\v');
// produces 4_294_983_168, or in binary:
//    00000000_00000000_00000000_00000000_
//    00000000_00000000_00000000_00000000_
//    00000000_00000000_00000000_00000001_
//    00000000_00000000_00111110_00000000 
```

Subsequently, to compute whether the given 127-bit character is in the set, you make sure it's less than `128` in value and then confirm whether the bit in the filter is set.

```swift
// is ' ' (value 32) in the 'set' 4_294_983_168?
bool has_space = set.contains(' ')
  = (32 < 128) & (set & (1ULL << 32))
  = (true) & (1)
  = 1; // true
```

This development  occurred in tandem with a realization I had made, and I was actually writing a response comment on the PR conversation thread about it when the change was submitted.

> Regarding adding `\v` and `\f` "because why not":
> > **Quoted Reply**: No, because it makes the filtering slower in this implementation. It's O(n) based on length.
> 
> Ah, very good point; hard to argue against that. However - since we are discussing speeds - if the _less common_ characters are put at the end of the char search string (like these ones which hardly ever show up, but may), then the nominal speed of a trim will actually be improved.
> 
> This is because the first time a **non-match** of the input char array is found, the `trim` loop breaks. Also, the `char_in_set` macro **short-circuits** when the first whitespace char is encountered.
> 
> So combining this means for the string `"[space][space][tab]abc def[space][newline]` it takes the same time to walk ` \t\n\r\f\v` as it would ` \t\n\r`.

---

In fact, the current orientation of the char array is not optimal.

Because of this, I'm actually going to rearrange the default `trim` strings to put space up front. I'll back this up with character frequency statistics.

---
### Summary
Closing thoughts.