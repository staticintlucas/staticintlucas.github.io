---
# empty front matter so Jekyll knows to process this file
---

# `stringslice`

`stringslice` is a simple Rust utility that provides Unicode-aware slicing of strings.

In Rust, all `String` and `&str` variables must contain valid UTF-8 data.
There are other types of string intended for interfacing with C and with the operating system, but for general text processing these 'regular' UTF-8 strings are used.

UFT-8 encodes Unicode code points using a variable width encoding.
The first 128 characters are encoded the same as their 7-bit ASCII representation.
This was chosen in order to maintain backwards compatibility, but it also reduces the memory required to store strings in English (or other languages using the Latin script) when compared to UTF-16 or UTF-32.
Other characters&thinsp;---&thinsp;such as accented letters, non-Latin alphabets, less common symbols, emojis, etc&thinsp;---&thinsp;use multiple bytes per code point.
The encoding is shown in the table below:

Range (hex)                           | Bytes (binary)
-------------------------------------:|:-------------------------------------
   `U+0000`&thinsp;--&thinsp;`U+007F` | `0XXXXXXX`
   `U+0080`&thinsp;--&thinsp;`U+07FF` | `110XXXXX 10XXXXXX`
   `U+0800`&thinsp;--&thinsp;`U+FFFF` | `1110XXXX 10XXXXXX 10XXXXXX`
`U+10000`&thinsp;--&thinsp;`U+10FFFF` | `11110XXX 10XXXXXX 10XXXXXX 10XXXXXX`

This encoding makes operations such as determining the number of characters in the string non-trivial.
With Rust's philosophy of providing zero-cost abstractions, there is no built-in method to perform these operations.
Luckily, for counting the number of characters there is a simple method.

~~~rust
let length = string.chars().count();
~~~

Here, the `chars()` method creates an iterator over UTF-8 code points.
The `count()` method takes characters from the iterator until it reaches the end, returning the count.
Of course, iterating over any sequence has complexity O(n), regardless of whether the data is UFT-8 encoded or not.
By forcing you to create an iterator explicitly, Rust makes it obvious that this is not a simple constant-time operation.

However, for taking a slice of a string Rust does allow indexing.
For example, you can run the following code:

~~~rust
let string = "abcdef";
let slice = &string[1..4];

println!("{}", slice);
~~~

This compiles and runs as expected, printing out the result:

~~~text
bcd
~~~

However, slicing a string like this uses *bytes* as indices, not code points.
In that example there were only ASCII characters, so each code point is only one byte.
On the other hand, consider this example where the string contains an emoji which is a single code point encoded as 4 bytes:

~~~rust
let string = "ab😎ef";
let slice = &string[1..4];
~~~

Running this produces the following result:

~~~text
thread 'main' panicked at 'byte index 4 is not a char boundary; it is inside '😎' (bytes 2..6) of `ab😎ef`'
~~~

The program attempted to slice the string in the middle of the UTF-8 sequence for '😎'.
This would have produced an invalid string.
Rust does check whether the resulting string would be valid; if the string is not valid it causes a `panic` and the program exits.

I created this project because I had fallen into this trap in the past.
The `stringslice` crate provides a `StringSlice` trait which allows a string to be sliced correctly regardless of the characters it contains.
Internally it uses a similar method to the `chars()` iterator I described previously.
It actually uses `char_indices()` to find the position (in bytes) of each character in the string.
Using those byte positions, it can correctly slice the string on the correct UTF-8 boundaries.

Here is the previous example, modified to use `stringslice`:

~~~rust
use stringslice::StringSlice;

let string = "ab😎ef";
let slice = string.slice(1..4);

println!("{}", slice);
~~~

This does not `panic`, and more importantly prints out the expected result:

~~~text
b😎e
~~~

Ultimately the project is pretty trivial; it's only around 50 lines of code if you exclude blank lines, comments, and unit tests.
However, I find it useful for my own projects, and I thought maybe someone else would too.

Source code is available on [GitHub], and the crate is available on [crates.io].

[github]: https://github.com/staticintlucas/stringslice
[crates.io]: https://crates.io/crates/stringslice
