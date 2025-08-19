---
tags:
  - c3
  - language-development
  - standard-library
title: 2025-08-18 - @reduce and @filter
permalink: work/2025-08-18
publish: true
---
#####  August 18th, 2025, ca. 22:00 -0400
## Today's Topics: Array `@reduce` and `@filter`
> [!info]- Article PGP Signature
>
> To verify this signature, use the public key on the [[Hello, World!|Home Page]] and follow the instructions linked there.
>
> ```
> -----BEGIN PGP SIGNATURE-----
> 
> iHUEABYKAB0WIQT2vFo+jLf+FWIQ2T8xrKfG/dldbgUCaKPwXAAKCRAxrKfG/dld
> biFIAQC8EDKWlMtSIEKf9G57rWXdczDAg0eboqLA3SjbVXijtgEAtAPSY6G0CxTm
> JAHNs2hxWmDC7DwZZU6l+R/GTHgcigg=
> =Tq0z
> -----END PGP SIGNATURE-----
> ```
>

### Today's Contributions
- PR: [c3lang#2419](https://github.com/c3lang/c3c/pull/2419)
- Issue: [c3lang#2418](https://github.com/c3lang/c3c/issues/2418)

### More work on arrays...
l've been on fire lately with these array-based core changes in the [C3](https://github.com/c3lang/c3c) standard library. First was the `@zip` work, then `range`, and now this; exciting! 

Today's work was the culmination of a branch I've been toying with in the background while cooking up `@zip` and the SHA-3/Keccak changes. I finally got around to honing everything, writing some unit tests, and all around _just sitting back in satisfaction_ of another incoming branch for C3.

Without further ado, let's talk about the main points of the [PR](https://github.com/c3lang/c3c/pull/2419).

##### `@reduce` and Friends
The `reduce` function, sometimes known as an array-based `accumulator` or `folding` operation, is used to iterate through an input array and _reduce_ its inputs down into a single, ==accumulated== value. Notice that "accumulated" is highlighted; this is important, because often in the nomenclature, the accumulator is also called the ==identity== value.

To see what it means to reduce, consider the mathematical function:
$$
\begin{eqnarray}
x = k + \sum_{i=0}^{n-1} a_i
 = \overbrace{a_0 + a_1 + \cdots + a_{n-1}}^{a}
\end{eqnarray}
$$

Honestly, it looks fancy just because I like writing it out, but all it's saying is "sum up all the values in the input array ***a***, add whatever the initial ***k*** value is, and assign that to ***x***", where ***x*** is the result of the reduction and ***k*** is an initial "accumulator" (or identity) value.

In this manner, we've ==reduced== all array values down to one from its constituent parts, by summing each value.

> [!example] `@reduce` in Action
> 
> ```swift
> Transaction[] xacts
> 	= server::fetch_xacts(user, yesterday, today_1300, limit: -1);
> int rounded_debits = array::@reduce(
> 	xacts,
> 	0,
> 	fn (i, e, u) => i + e.is_debit ? math::ceil(e.cash_out) : 0
> );
> io::printfn(
> 	"%s has spent ~$%d between %s and %s.",
> 	user.name, rounded_debits, yesterday, today_1300
> );
> ```
> We quickly used a lambda function to __reduce__ all transactions within the time period to a single summed value, starting from `$0` (the second parameter, _k_ from the equation).
> 
> If you were thinking about using a `@filter` macro on the array first to filter out any non-debit transactions, _you'd be right_! That is the ideal and more legible process.
> 
> Note that this isn't your ordinary `@sum` macro, even though that was added. This is because the lambda is using the `math::ceil` function to round up on each debit to the bank account; something that `@sum` will not do on its own.

The `@reduce` function also offers some pretty cool and optional "summary operators" which I've built into the stdlib call. If the `#operation` (lambda) is a string type, it will be checked against a table at compile-time. Here are a few summary operators that I added in:
- `"+"`: of course, as a "sum" operator.
- `"^"`: a chaining XOR operator for each array element.
- `"&~"`: a series of anti-masking operations.
- `"count_positive"`: counts how many numeric inputs are positive values.
- ... and quite a few more.

These are used in the following way:
```swift
uint initial_flags = app::read_flags_register();
uint unset_flags = { 0x0001, 0x8074, 0x1001 };   // we don't want these on!
uint without_flags = array::@reduce(unset_flags, initial_flags, "&~");
```
Now `without_flags` will be equal to `initial_flags & ~unset_flags[0] & ~unset_flags[1] & ~unset_flags[2]`, which should unset all undesired flags from the read register value.

Easy!

##### `@filter` and Such
The `@filter`, `@any` and `@all` functions are so-called "predicate" based functions which can use a truth statement to filter out or analyze elements across the range of the input array.
- `@filter` returns a shallow-copy array of each input array element matching -- i.e., returning `true` when input through -- the `#predicate` function.
- `@any` returns `true` if __any__ of the array's elements satisfies the `#predicate` function.
- `@all`, as the name suggests, only returns `true` when __all__ array elements satisfy the `#predicate` function.

> [!example] Using `@filter` for Online Users
> 
> ```swift
> @pool()
> {
> 	List {User*} online_users = server::get_online_player_handles(instance, max: 150);
> 	User*[] above_level_50 = array::@tfilter(online_users, fn (e, u) => e.level > 50);
> }
> ```
> The `get_online_player_handles` function returns a List of pointers to a maximum of 150 online players. That List is then given the the `@tfilter` macro (`@filter` using the temp allocator), which returns only the `User` handles with a player level above `50`.
> 
> Just from this brief example, you can understand the power of this macro and its analogs.

> [!example] Using `@any` for Bulk Flagging
> 
> ```swift
> EmailMeta[] checked = server::blocking_threaded_scan(mail_pool);
> if (array::@any(checked, fn (e, u) => e.spam_score > 5.0))
> {
> 	mail::dispatch("admin@site.net", "Subject: [SPAM] detected", ...);
> 	// ...
> }
> ```
> Perhaps not the greatest example (I'm making these up on the fly here!), but `@any` here is used to detect whether the bulk of emails being processed asynchronously came back flagged as spam.
> 
> The nice thing about `@any` and `@all` is that they short-circuit, meaning the entire array won't be scanned when a matching condition is found early. For `@any`, it stops processing at the first match of the `#predicate` lambda. For `@all`, it stops at the first `false` return of `#predicate`, if any.

These are just a few quick examples of the many, many possibilities for these array-based macros in the C3 standard library. I really hope to see them used in the future!

### Fun with compile-time lambdas
The [issue with the compiler](https://github.com/c3lang/c3c/issues/2418) that I discovered today resulted in a failure of the compiler's awareness of function pointers underlying compile-time constants. If you read over the issue, you'll see that triggering it is based on whether the `$func` constant is being assigned to the given `#operation` expression:
> [!faq] Problematic Code
> 
> Simple uncomment `// = #operation;` to fix the compiler assertion. `$func` is a null function pointer and obviously cannot be applied with `identity` and `element` at compile-time if it doesn't point to anything.
> 
> ```swift
> import std::io;
> macro @reduce_fn(#array, #identity) @const
> {
> 	return @typeid(fn $typeof(#identity) ($typeof(#identity) i, $typeof(#array[0]) a) => i);
> }
> 
> macro @do_thing(array, identity, #operation)
> {
> 	// the problem line...
> 	$typefrom(@reduce_fn(array, identity)) $func;// = #operation;
> 	foreach (element : array) identity = $func(identity, element);
> 	return identity;
> }
> 
> fn void main()
> {
> 	int[] arr = { 0, 7, 20, -4 };
> 	int reduced = @do_thing(arr, -1, fn (a, b) => a * (b + 3));
> 	assert(reduced == 690);
> 	io::printfn("reduced: %d", reduced);
> }
> ```

In typical C3 fashion, this bug was fixed (not by me) _only about 2 hours_ after it was submitted. ==Lightning speed!==

### Summary
Today I was really hoping to get some more work done with my UEFI code. After a brief chat with some community members, it seems I'll need to dig a bit further into trying to link the resulting EFI binary with `c3c` instead of calling out to `clang` and `lld-link` at the end of the C3 artifact compilation step.

Anyway, today was a pretty huge expenditure of mental energy on all this stuff. I may keep it a tad lighter tomorrow for the sake of maintaining a day-job. :)

Be seeing you.