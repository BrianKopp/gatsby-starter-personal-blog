---
title: Rust Keyword Counter
subTitle: A basic command line tool
category: rust
cover: cuddlyferris.png
---

I've been hearing more and more about this trendy language Rust.
"It's great" they said, "It'll be fun" they said. Well,
yes and no. The idea of Rust is fantastic. A machine-language-compiled,
strongly-typed, memory-safe, concurrency-safe, lightning fast
programming language **does indeed sound fantastic**, and it is, don't get me wrong.

There is, however, a decent price to pay in learning curve.
You're not going to come in, learn some basic syntax, and start
shredding up the web after a few days of fiddling like you did when
you learned python. Rust is tough. You need to
[read the book](https://doc.rust-lang.org/book/). The ownership,
borrowing, and implementation details are probably different
than what you've used before. That was certainly true for me.

In this article, I'll be making a simple command line utility that
takes a file as an argument, and then reports out the most frequent
words in the file and their occurences in order. Pretty
straightforward, right? Ok, let's hop in.

## Setup

```bash
cargo new keyword
cd keyword
```

This results in a project structure like:

```text
|- .gitignore
|- Cargo.toml
|- src/
   |- main.rs
```

Checking out the main file...

```rust
fn main() {
    println!("Hello, world!");
}
```

It's pretty straightforward, obviously. Let's go ahead and run
our hello-world app using `cargo run`.
This will trigger a build, adding a Cargo.lock file, and then we see
`Hello, world!` in the output.

## Getting Command-Line Arguments

We're taking a single command line argument, so let's start by printing that out.
In the `src/main.rs` file we'll get command line arguments from the `std::env`
library's `args()` function. Update your file to look like this:

```rust
use std::env;

fn main() {
    let args: Vec<String> = env::args().collect();
    println!("{:?}", args);
}
```

`args()` returns an iterator, and since iterators are evaluated lazily in Rust,
you have to `collect()` them to pull them into the vector of strings. Running
`cargo run` on this file outputs the following text:
`["target\\debug\\keywords.exe"]`. Even though we didn't pass any command line
arguments, Rust uses C-style command line arguments where the first argument is
the location of the executable. If you're on mac or linux, this path will
obviously look slightly different.

Let's try passing in a filename to this program. Try `cargo run text.txt`.
You'll see `["target\\debug\\keywords.exe", "text.txt"]`. Good, we're
getting the command line arguments coming through.

## Reading a File

In order to count keywords, we're going to need to read a file. Add `text.txt`
to your project root with the following text:

```text
On the other hand, we denounce with righteous indignation
and dislike men who are so beguiled and demoralized by the
charms of pleasure of the moment, so blinded by desire,
that they cannot foresee the pain and trouble that are bound
to ensue; and equal blame belongs to those who fail in their
duty through weakness of will, which is the same as saying
through shrinking from toil and pain. These cases are
perfectly simple and easy to distinguish. In a free hour,
when our power of choice is untrammelled and when nothing
prevents our being able to do what we like best, every
pleasure is to be welcomed and every pain avoided. But in
certain circumstances and owing to the claims of duty or
the obligations of business it will frequently occur that
pleasures have to be repudiated and annoyances accepted.
The wise man therefore always holds in these matters to
this principle of selection: he rejects pleasures to secure
other greater pleasures, or else he endures pains to avoid worse pains.
```

In the `main.rs` file, import `std::fs` and add the following
beneath the existing code.

```rust
let content = fs::read_to_string(&args[1]).expect("should have been able to read file");
println!("{}", content);
```

Now when you call `cargo run text.txt`, it should print out the file's
contents.

If you call `cargo run text2.txt`, to a file that doesn't exist, Rust will
panic out on you.

```text
thread 'main' panicked at 'should have been able to read file: Os { code: 2, kind: NotFound, message: "The system cannot find the file specified." }', src\libcore\result.rs:997:5
note: Run with `RUST_BACKTRACE=1` environment variable to display a backtrace.
error: process didn't exit successfully: `target\debug\keywords.exe text2.txt` (exit code: 101)
```

Let's make this a little nicer. If someone is using our binary, our error message
should be succinct. Update your main.rs file to the following:

```rust
use std::env;
use std::fs;
use std::process;
use std::io;

fn main() {
    let args: Vec<String> = env::args().collect();

    let file_name = &args[1];
    let file_content_result = fs::read_to_string(file_name);
    if let Err(err) = file_content_result {
        match err.kind() {
            io::ErrorKind::NotFound => println!("file '{}' not found", file_name),
            _ => println!("unexpected error occurred: {}", err),
        }
        process::exit(1)
    }
    let contents = file_content_result.unwrap();
    println!("{}", content);
}
```

There's a lot going on in here, especially if you're new to Rust. Let's go
line by line.

Line 10, `fs::read_to_string(file_name)` returns a `Result<String>`. Result
is a common enum in Rust, with values Ok<T> or Err. By returning this enum,
Rust encourages exhaustive coverage of possibilities, success and failures.

In line 11, pattern matching is used to check if `file_content_result`, which
is a type of the `Result<String>` enum, is of the Err enum member, with *some*
error. If it does match, then `err` then gets assigned to the error that was
returned. Once we get in that block, use a match block to report a meaningful
error message to the user. If the `err.kind()` is NotFound, a very common situation,
I want to give a meaningful error message. Otherwise, I'll just print the error
message in all its gory details. `match` is kind of like switch in other languages,
except that it **requires options be exhaustive**. You are allowed to use
`_` to denote "everything else". Finally, in this block, I `process::exit(1)`, just
to return a non-zero exit code. No return statements are required here in rust.
Rust interprets an expression that is not terminated with a semicolon to indicate
a return.

If `let Err(err)` doesn't match `file_content_result`, which would be the case if
it is the other enum value `Ok`, we can go ahead and `unwrap()` the `Result<String>`
to get its `String` value, and print it.

## Getting the Words

If you are like me, and don't super-grok regular expressions, I'm sorry.

We're going to update our toml file to include regex. Add `regex = "1"`
to your `[dependencies]` section of your `Cargo.toml` file, and then run
`cargo build`.

We'll create a regular expression, and then find all its matches in
the file's contents.

```rust
// ...
extern crate regex;
use regex::Regex;
// ...
    let contents = file_content_result.unwrap();
    // create regular expression
    let re = Regex::new(r"[a-zA-Z_$][a-zA-Z_$0-9]*").unwrap();
    for content_match in re.find_iter(&contents) {
        println!("{}", content_match.as_str());
    }
```

Basically, this will match any word that begins with an alphabetical
character, the underscore `_` character, or the dollar sign `$` character,
and followed by at least zero other characters that are
either alphabetical, numerical, `_` or `$`.
This is probably a good starting point for finding variable names.

We go ahead and unwrap() this, not expecting any errors.

Next, we'll find matches and iterate over them, and printing the result.
`content_match` is of `Match` type, so to get the text that matched,
we have to call `.as_str()` to get the matching text.

Now, when we call `cargo run text.txt`, we'll get a lot of output!

## Counting Hits

The next step in our journey requires counting matches of words. For
that, we'll use a `HashMap` structure from the `std::collections` library.

```rust
// ...
use std::collections::HashMap;
// ...
    let re = Regex::new(r"[a-zA-Z_$][a-zA-Z_$0-9]*").unwrap();
    let mut counts_by_word: HashMap<&str, u32> = HashMap::new();
    for content_match in re.find_iter(&contents) {
        let count = counts_by_word
            .entry(content_match.as_str())
            .or_insert(0);
        *count += 1;
        println!("{} - {}", content_match.as_str(), count);
    }
```

Once again, there's a lot going on in here. We initialize
`counts_by_word` as a `mut` mutable object, since we'll be, well,
mutating it.

In the for-loop, there is a multi-part statement. We grab the `.entry`
on the HashMap at the key corresponding to the matched string. If that
entry is not yet in the HashMap, we `.or_insert(0)` with the value zero.
This then returns the existing value, or new value of zero
stored in the HashMap for that key. Next, we use the dereferencing
operator on count to increment it.

Running `cargo run test.txt` now results in:

```text
On - 1
the - 1
other - 1
hand - 1
--- truncated ---
to - 9
secure - 1
--- truncated ---
pains - 1
to - 10
avoid - 1
worse - 1
pains - 2
```

Notice the two entries shown for `to`. The count is indeed incrementing!

## Sort and Print Results

At this point, we have a HashMap whose keys are words and value is the
number of times the word is mentioned. The last piece we want to
do is to sort the results and display the most frequent words at the
bottom. To do this, we'll copy the results into a vector and then sort it.

```rust
println!("results:");
let mut sorted_words: Vec<_> = counts_by_word.iter().collect();
sorted_words.sort_by(|a, b| a.1.partial_cmp(b.1).unwrap());
for result in sorted_words.iter() {
    println!("{} - {}", result.0, result.1);
}
```

Here, we iterate and collect the counts_by_word HashMap into a
vector of key, value tuples. Then, we call `.sort_by()` which
sorts the vector in-place using a closure to evaluate the sort
result. We sort on the count, which is the second member of
each tuple, using the `.partial_cmp()` function, unwrapping the
result since we don't expect any craziness there.
Finally we print out the results like `{key} - {value}`.

Let's run it using `cargo run text.txt`! Now you should see the
results printed in order.

Here is the final `main.rs` file.

`gist:43da8bfc760cf1909008f1a3fd5936f0#main.rs`

## Wrapping Up

We now have a working Rust command line tool. Build it using
`cargo build --release` and put it in your path to use it!

Thanks for stopping by!
