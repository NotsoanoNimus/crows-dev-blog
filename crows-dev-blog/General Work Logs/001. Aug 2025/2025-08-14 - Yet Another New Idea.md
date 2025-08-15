---
tags:
  - c3
  - language-development
  - hashing
  - hookshot
  - ideas
title: 2025-08-14 - Yet Another New Idea
permalink: work/2025-08-14
publish: true
---
#####  August 14th, 2025, ca. 22:00 -0400
## Today's Topics: Merging & an Idea
> [!info]- Article PGP Signature
>
> To verify this signature, use the public key on the [[Hello, World!|Home Page]] and follow the instructions linked there.
>
> ```
> -----BEGIN PGP SIGNATURE-----
> 
> iHUEABYKAB0WIQT2vFo+jLf+FWIQ2T8xrKfG/dldbgUCaJ6hUwAKCRAxrKfG/dld
> blzmAQCyOdR90ILahbgXJ2Lj3kJEuXWfE7g5J0UwbT4KGo6DlwD+KEklqx2ii2Xg
> lAqY98NgpF7KZEIFE3Fr7gtB8JLlJgY=
> =CnQh
> -----END PGP SIGNATURE-----
> ```
>

### Today's Contributions
- New Repo: [hookshot-ftp](https://github.com/NotsoanoNimus)
- PR: [c3lang#2403](https://github.com/c3lang/c3c/pull/2403)

### Revisiting `a5hash` and Friends
Sometime last month, I submitted a [rather large addition](https://github.com/c3lang/c3c/pull/2293) to the C3 standard library which was aimed at providing a new, more modern suite of non-cryptographic data hashing methods to employ in various projects.

This addition included the following hash types:
- [wyhash2](https://github.com/JackThomson2/wyhash2)
- [MetroHash64/128](https://www.jandrewrogers.com/2015/05/27/metrohash/)
- [a5hash](https://github.com/avaneev/a5hash) & [komihash](https://github.com/avaneev/komihash)

What I didn't realize when replacing the old [FNV hashing](https://en.wikipedia.org/wiki/Fowler%E2%80%93Noll%E2%80%93Vo_hash_function) was that I failed to also update the compile-time builtin `$$str_hash` to use `a5hash` like the `String.hash()` method. This meant the following assertion would unknowingly fail:
```cpp
assert("hello, world".hash() == $$str_hash("hello, world"));
```

I awoke to a request from C3's creator to please fix this discrepancy, which meant implementing `a5hash` in pure C. After doing so in [this pull request](https://github.com/c3lang/c3c/pull/2403), I added the following test function to the standard library's tests to ensure this doesn't lose synchronization again if either's hash function changes again:
```cpp
fn void test_builtin_string_hashing() => @pool()
{
	var $x = "";
	ulong l;
	// Hash strings "0", "a1", "aa2", "aaa3", "aaaa4", ...
	$for var $i = 0; $i < 65; ++$i:
		l = string::tformat("%s%s", $x, $i).hash();
		var $r = $$str_hash(@sprintf("%s%s", $x, $i));
		assert((uint)l == (uint)$r, "Builtin $$str_hash mismatch against String.hash()");
		$x = $x +++ "a";
	$endfor
}
```

### Hookshot: My New Idea
While the name sounds cool and nostalgic, this project idea will probably leave a lot of security-focused readers _uncomfortable_, to say the least. Check out [the README](https://github.com/NotsoanoNimus/hookshot-ftp) for the repository, take a deep breath, and continue.

> [!tip] Confession
> I've always been curious about exposed FTP servers with anonymous logins. When I was younger, this was a frequent fixation of a few \[terrible-but-teaching\] scripts that took me from **non-functional** to **barely duct-taped**, but it was a step up nonetheless.
> 
> So, I thought, _why not take this old friend to a new height_? Why not put to use some of the skills I've gained in the last decade and brush up on this idea?

I genuinely think the curiosity about what's "out there" and public is a ==good thing== to explore. At least for me, this curiosity always activated some sort of _treasure-discovery_ reward system in my mind, because - really - you never knew what payload you were going to strike next. This made learning **exciting** as well.

Most of the time, data from anonymous FTP servers tends to be someone's web-root folder with nothing in it, but occasionally you get the collection of pirated movies, and not much else. This is probably still a moral gray-area, given the amount of people setting up FTP servers for hosting deeply personal data, with _no clue_ what they're doing - but I like to justify it this way:

1. "Anonymous" access servers, by their definition, _authorize_ access to a server's contents by the mere fact that anyone can access them **without credentials**, and
2. It is ultimately the **responsibility of hosts** (read: users) to secure their own data on the public internet, and to be receptive to notifications that their data may be compromised (`spoiler alert: Hookshot provides such a notification by default`).

Whether this idea is actually acceptable beyond my own opinion will be determined by any feedback I receive, and I look forward to that.

### Brief Housekeeping & Summary
Today, I got [[2025-08-13 - Delicate Balancing Acts|yesterday's]] PRs mostly cleaned up and merged (or at least ready for a merge). There was, as anticipated, no movement on UEFI stuff today.

What wasn't anticipated was that there was no movement on my SHA-3 submission either. Instead, I become focused on my shiny new project, which I'll be adding into my regularly rotating cycles of interest and commitment.

Onward to Friday!