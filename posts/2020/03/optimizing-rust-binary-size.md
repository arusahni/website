<!--
    .. title: Optimizing Rust Binary Size
    .. slug: optimizing-rust-binary-size
    .. date: 2020-03-30 20:00:05 UTC-04:00
    .. tags: rust, code
    .. link:
    .. description: In which I steadily reduce the size of the git-req Rust binary by applying various optimizations.
-->

I develop and maintain a git extension called
[git-req](https://arusahni.github.io/git-req/). It enables developers to check
out pull requests from GitHub and GitLab by their number instead of branch
name. It initially started out as a bash script that invoked Python for harder
tasks (e.g., JSON parsing). It worked well enough, but I wanted to add
functionality that would have been painful to implement in bash.  Additionally,
one of my goals was to make it as portable as possible, and requiring a Python
distribution be available flew against that. That meant that I needed to
distribute this as a binary instead of a script, so I set about finding a
programming language to use. After surveying what was available, and
determining what would be the best addition to my toolbox, [I selected
Rust](https://www.rust-lang.org/).

The programming language has a steep learning curve, but has been fun to learn
and immerse myself within.  The community is great, and I'm excited to find
more opportunities to use Rust in the future.

The rewrite took a while to accomplish, but when all was said and done,
everything worked, and worked well.  I was able to implement some snazzy new
features as well as polish some rough edges. However, for how "simple" I felt
the underlying program to be, it clocked in at 13 megabytes. That felt like a
lot.  So, I decided to see what could be done.

For those playing along at home, the starting binary size was: **13535712
bytes (12.9MB)**.

## Phase 1: Building

The first thread I pulled was ensuring that the compiler would output code in
such a way that it prioritized disk space over speed. I'm fine with the build
taking slightly more time, as well as with the program being slightly slower
- most commands incur network traffic, so a few extra milliseconds of
CPU time are nothing in comparison. I found two simple additions to my
`Cargo.toml` got me all I needed:

#### 1. Optimization Level

The optimization level instructs the compiler as to what trade-offs it should
make at compile time. One can opt for longer compile times and larger file size
in exchange for faster run times, or instead request a smaller file size for
longer compile times and slightly slower run times. To turn this knob, add the
following to `Cargo.toml`:

```toml
[profile.release]
opt-level = "s"
```

[Possible `opt-level` values
include](https://doc.rust-lang.org/cargo/reference/profiles.html#opt-level):

* `0`: no optimizations
* `1`: basic optimizations
* `2`: some optimizations
* `3`: all optimizations
* `"s"`: optimize for binary size
* `"z"`: optimize for binary size, but also turn off loop vectorization.

The docs encourage experimentation - I strongly suggest heeding that guidance.
My initial guess was `"z"`, which seemed like the most extreme option.  After
testing all possible values, it turned out `"s"` resulted in the smallest
binary size.

New binary size: **12832464 bytes (12.2MB)**.

#### 2. Link Time Optimization (LTO)

Link Time Optimization is an optimization phase that the compiler carries out
where it assesses the entire program (instead of an individual file) to
determine if there are optimizations to be made (e.g., removing dead code).  To
enable it, add the following to `Cargo.toml`:

```toml
[profile.release]
lto = true
codegen-units = 1
```

This instructs the Rust compiler to apply a "full" set of optimizations _only_
when building for release. [Possible `lto` values
include](https://doc.rust-lang.org/cargo/reference/profiles.html#lto):

* `false`: LTO only across the crate or its codegen units.
* `true` or `"fat"`: LTO across _all_ crates in the dependency graph.
* `"thin"`: similar to `"fat"`, but faster to run while offering _similar_
  gains to `"fat"`.
* `"off"`: No LTO

The `codegen-units` setting limits how many pieces the compiler may split the
crate into in order to optimize build parallelization. One of the great things
Rust's borrow checker enables is fearless parallelization, which it and its
tooling exploit. By setting this value to `1`, I was able to ensure that the
linking phase would not parallelize, and instead consider the full codebase,
thus ensuring that the code was properly optimized (at the expense of longer
compilation times).

New binary size: **8338640 bytes (8.0MB)**.

62% of the original binary size. Nice, but why stop there?

## Phase 2: Trimming the Fat

Now that we've made the compiler play nicely, where else can we get some gains?
Since Rust ships with a fairly minimal standard library, developers rely on its
robust package ecosystem for things like JSON serialization and HTTP requests.
One issue with this is that external dependencies are the primary vector for
bloat in any application. If only there were a way to measure such bloat in a
Rust application...

... oh wait, there is: [cargo-bloat](https://github.com/RazrFalcon/cargo-bloat).

Running it against git-req with the `--release --crates` flags outputs:

```
 File  .text     Size Crate
 4.4%  11.3% 360.4KiB reqwest
 3.8%   9.6% 306.0KiB std
 3.3%   8.4% 267.5KiB clap
 3.2%   8.1% 259.5KiB regex
 2.9%   7.5% 237.9KiB regex_syntax
 2.6%   6.5% 208.3KiB [Unknown]
 2.4%   6.1% 193.3KiB rustls
 1.3%   3.2% 103.1KiB goblin
 1.2%   3.2% 101.3KiB backtrace
 1.2%   3.2% 100.4KiB libgit2_sys
 1.1%   2.9%  93.3KiB yaml_rust
 1.1%   2.7%  86.6KiB git_req
 1.1%   2.7%  85.8KiB ring
 1.0%   2.7%  84.6KiB unicode_normalization
 0.9%   2.3%  74.0KiB object
 0.9%   2.2%  70.0KiB h2
 0.7%   1.7%  55.0KiB hyper
 0.5%   1.3%  41.6KiB http
 0.5%   1.3%  41.5KiB duct
 0.5%   1.3%  40.1KiB term
 4.4%  11.3% 361.4KiB And 79 more crates. Use -n N to show more.
39.1% 100.0%   3.1MiB .text section size, the file size is 8.0MiB
```

Wow - git-req (`git_req`) only accounts for 1.1% of the file, with a long tail
of crates bringing up the rear. More interestingly, there are a few crates that
dominate the file size. Let's tackle the big one: `reqwest`.

As someone who does a lot of Python development, when I first started with Rust
I wanted something that mimicked the ergonomics of the popular Requests
library. The [phonetically-similar reqwest offers just
that](https://docs.rs/reqwest/). Unfortunately, it was pretty big, and there
appeared to be a lot of the library that I wasn't using, nor was I planning on
using. With those two points, I recognized that this library was a prime
candidate for replacement.

I started with [this
post](https://medium.com/@shnatsel/smoke-testing-rust-http-clients-b8f2ee5db4e6)
that discussed the merits of various HTTP crates available to developers. Of
importance to me were those that had: rustls support, serde support, minimal
use of unsafe, and a sane API. Based on those criteria, [`ureq` checked all the
boxes](https://docs.rs/ureq/).

Replacing reqwest with ureq was fairly straightforward.  Let's see how this
manifests in file size...

```
File  .text     Size Crate
 3.9%  10.5% 267.5KiB clap
 3.8%  10.1% 258.2KiB regex
 3.5%   9.3% 237.2KiB regex_syntax
 3.4%   9.1% 231.3KiB std
 3.1%   8.2% 208.2KiB [Unknown]
 2.7%   7.2% 182.4KiB rustls
 1.5%   4.0% 103.1KiB goblin
 1.5%   4.0% 101.4KiB backtrace
 1.5%   3.9% 100.4KiB libgit2_sys
 1.4%   3.8%  95.9KiB yaml_rust
 1.3%   3.6%  91.6KiB git_req
 1.2%   3.3%  84.6KiB unicode_normalization
 1.2%   3.2%  82.3KiB ureq
 1.1%   2.9%  74.0KiB object
 1.1%   2.8%  71.8KiB ring
 0.6%   1.6%  41.6KiB duct
 0.6%   1.6%  40.0KiB term
 0.6%   1.6%  39.9KiB url
 0.6%   1.5%  37.7KiB time
 0.4%   1.0%  26.7KiB rustc_demangle
 2.3%   6.2% 158.7KiB And 43 more crates. Use -n N to show more.
37.4% 100.0%   2.5MiB .text section size, the file size is 6.6MiB
```

Wow, the piece of functionality that was the biggest offender is now not even
in the top 10.

Binary size: **6967472 bytes (6.7MB)**.

## Phase 3: Things I'm Not Comfortable Doing Yet

In my research, I stumbled upon some optimizations that I wasn't comfortable
applying yet - mostly because I want to spend some time to ensure there won't
be any runtime implications for git-req.

#### 1. Stripping

Most rendered binaries have to ship with content to support all possible
use-cases for the application. If you know how a binary will be used, you can
apply post-processing to it to strip out the unnecessary portions. The [strip
tool](https://en.wikipedia.org/wiki/Strip_%28Unix%29) is one of the more
popular utilities that does this. Applying it to git-req yields substantial
savings: **4718240 bytes (4.5MB)**! Why wouldn't I want to ship this
immediately? One word: backtraces.

When a release-grade Rust application panics, if the `RUST_BACKTRACE`
environment variable is set to `1`, the application will print out a backtrace
before it dies. This is immensely useful for debugging, and, given the amount
of variance in the environments this application is running, I feel that
playing file size golf at the expense of supportability is out of the
question... for now.

#### 2. Features

[Rust has the concept of
"Features"](https://doc.rust-lang.org/cargo/reference/features.html), which
allow developers to explicitly modify parts of the application at compile-time.
In the case of git-req, the `color-backtrace` library is especially useful to
me, the program's author, because scrutinizing backtraces is a regular part of
my workflow. Hopefully, this is significantly less of a problem for end-users,
so whatever benefit they may get is minimal, at best. I could update this to be
hidden behind a feature flag, enabling me to not ship the library.  Given it
isn't in the top 100 contributors to bloat in git-req, I consider if not worth
the effort to implement and maintain.

## In Closing

Question everything - gains are to be had. [Check out
`git-req`](https://arusahni.github.io/git-req)!
