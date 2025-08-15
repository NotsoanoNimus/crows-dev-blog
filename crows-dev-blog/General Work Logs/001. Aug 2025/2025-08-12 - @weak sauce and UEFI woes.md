---
tags:
  - c3
  - uefi
  - web
  - osdev
  - language-development
title: 2025-08-12 - @weak sauce and UEFI woes
permalink: work/2025-08-12
publish: true
---
#####  August 12th, 2025, ca. 14:00 -0400
## Active Projects: this site, [xmit.xyz](https://xmit.xyz/), and [uefi.c3l](https://github.com/NotsoanoNimus/uefi.c3l)
> [!info]- Article PGP Signature
>
> To verify this signature, use the public key on the [[Hello, World!|Home Page]] and follow the instructions linked there.
>
> > [!warning] Updated Signature Timestamp
> > In order to significantly shorten my signature blocks, I've opted to create _detached signatures_ instead of compressing and using entire articles. I should have done this to begin with.
> > 
> > This means the timestamp within this signature may not exactly match the article date, but it will verify.
> 
> ```
> -----BEGIN PGP SIGNATURE-----
> 
> iHUEABYKAB0WIQT2vFo+jLf+FWIQ2T8xrKfG/dldbgUCaJydVAAKCRAxrKfG/dld
> bmhgAP0fQj4CzLYCJaW0tUBuQs4fcHPK4d+JS7r3TeZKZg/khAD/a65EUHGsnvHz
> Edy8VGAKsRY3PRqAe9z3n8BsczZY+ws=
> =D2cY
> -----END PGP SIGNATURE-----
> ```

### UEFI Project
I've been working on and off on my own [GNU-EFI analog](https://github.com/NotsoanoNimus/uefi.c3l) built exclusively in [C3](https://c3-lang.org/), and linked to resulting EFI applications with a little help from our friend `clang`. I'll show the linker command as work on this project progresses, of course, since it does essentially nothing right now.

Attempting to recreate _gnuefi_ has been - needless to say - a challenge so far. There are multiple things working against me constantly:
- Working with a new programming language
  - _Meaning there are quite a few bugs to work out still and I never know where to point my finger for any given issue_
- Needing to stage the linking separately from the compilation of the objects
- Forming my own independent "architecture" and runtime for UEFI applications to use
- Attempting to integrate C3's standard library into the mix

#### @weak
Enter the first and primary issue of the day: the `@weak` symbol.

###### lib/std/libc/libc.c3
```c
module libc @if(env::NO_LIBC);
// ...
fn void* malloc(usz size) @weak @extern("malloc") @nostrip
{
	unreachable("malloc unavailable");
}
fn void* calloc(usz count, usz size) @weak @extern("calloc") @nostrip
{
	unreachable("calloc unavailable");
}
fn void* free(void*) @weak @extern("free")
{
	unreachable("free unavailable");
}
fn void* realloc(void* ptr, usz size) @weak @extern("realloc") @nostrip
{
	unreachable("realloc unavailable");
}
```

Notice the attributes to the right? These are [weak declarations](https://en.wikipedia.org/wiki/Weak_symbol) for expected _libc_ memory functions which attempt to point themselves to other externally-defined symbols by the same name.

This means if your implementation has its own `malloc`, `calloc`, etc. export, then it should overwrite these symbols upon linking.

However, it seems C3's `@weak` attribute isn't properly passing this property onto the generated `windows-x64` target executable. This causes a slew of `duplicate symbol` errors upon linking. Indeed, more research seems to point to _weak_ symbols being more exclusive - or at the very least native - to the ELF executable format.

Nonetheless, a [GitHub Issue](https://github.com/c3lang/c3c/issues/2396) has been created to hopefully remediate the issue at the language level. We'll see what happens!

#### Thread-Local Storage
I'm only just now scratching the surface of this, but... [Thread-Local Storage](https://stackoverflow.com/questions/35692188/what-is-thread-local-storage-why-we-need-it) is an obscure demon that's haunting my ability to link the C3 standard library with my EFI applications.

Why might I care about this for a UEFI application, you might ask.

Well, if one would like to use the C3 standard library with `NO_LIBC`, one must be able to use the native (i.e., non-POSIX) threading interface, which is essentially unimplemented and returns mostly `NOT_IMPLEMENTED` faults.

But even if I explicitly **DID NOT** care about multiprocessing with UEFI and explicitly opted out, the presence of `tlocal` storage specifiers on variables like `_errno_c3` requires at least a dummy declaration of a `_tls_index` symbol.

###### lib/std/libc/os/errno.c3
```c
module libc::os @if(env::NO_LIBC || !env::HAS_NATIVE_ERRNO);

tlocal int _errno_c3 = 0;

fn void errno_set(int err) => _errno_c3 = err;
fn int errno() => _errno_c3;
```

Right, this can't be changed at the moment, for obvious reasons: the last thing I'm going to commit to doing is change the standard library for the sake of UEFI alone.

This has been filed into my ==TODO== list to learn more about, for sure.

### Summary
After these two woes, my output today was abysmal. Almost no code was created toward anything, because the whole time was spent futzing around with duplicate symbols and missing C runtime (CRT) features.

Hoping for a fresh start tomorrow, and to actually progress this C3-based framework (and thus bootloader) a bit. I'll start moving in this direction even if there is no movement on getting the C3 standard library to function.

See you tomorrow!