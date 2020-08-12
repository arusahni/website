<!--
    .. title: Kicking the Tires with Elixir (Part 1)
    .. slug: elxir-intro-01
    .. date: 2020-08-12 17:41:11 UTC-04:00
    .. tags: tech, elixir
    .. link: 
    .. description: In which I jump into Elixir pipes
    .. type: text
-->

Lately I've been playing with Elixir, a functional language that sits over
[Erlang](https://www.erlang.org/) and
[the OTP framework](https://learnyousomeerlang.com/what-is-otp). It's exciting
because it handles embarrassingly scalable problems with aplomb, enabling a
high level of parallelism and concurrency with a great developer experience.
While I think Rust shines at the system programming level, Elixir seems like a
perfect candidate for web services - balancing power with ergonomics.

There are plenty of well-written posts about the language and its associated
libraries. I wanted to touch on what stood out to me as a Pythonista, web
developer, and nerd. I don't know how many posts will comprise the series, but
I have at least a few topics in mind.

First up: pipes.

# Pipe ALL the Things

Look at any Elixir code and one of the first things you'll stumble upon is
`|>` - the pipe operator. It's syntactic sugar for passing the result of the
previous statement in the pipeline into the subsequent partial as the
first parameter.  This feels similar to the convention in Kotlin where
functions passed in at the last position can be written outside of the
parentheses, facilitating a DSL-like expression.

Why is this good? It incentivizes you to group your code into logical blocks
and remove temporary variables. Here's a contrived example where I have a
function that accepts a collection of items, applies some transformation,
filtering, and validation, and then processes the end result.


```elixir
def add_widgets(widgets) do
    trimmed = trim_widget_names(widgets)  # remove extraneous whitespace
    deduplicated = Enum.uniq_by(trimmed, fn %{name: val} -> val end)
    validated = Enum.filter(trimmed, fn %{name: val} -> String.length(val) > 4 end)
    insert(validated)
end
```

Look at all those unnecessary variables! Did you catch the bug? On line 4, I
accidentally referenced the wrong variable: it should be `deduplicated`. This
sort of error is super-sneaky. Since we're composing these functions, we can
rephrase this as:

```elixir
def add_widgets(widgets) do
    insert(Enum.filter(Enum.uniq_by(trim_widget_names(widgets), fn %{name: val} -> val end), fn %{name: val} -> String.length(val) > 4 end))
end
```

Hard to read and refactor, right? Which args go with which function? Maybe some
indentation can help.

```elixir
def add_widgets(widgets) do
    insert(
        Enum.filter(
            Enum.uniq_by(
                trim_widget_names(
                    widgets
                ),
                fn %{name: val} -> val end
            ),
            fn %{name: val} -> String.length(val) > 4 end
        )
    )
end
```

A little more readable, but it's still hard to update - I'd need to parse
levels of indentation, and inserting a function in the middle results in
re-jiggering everything. This also triggers some repressed memories of callback
hell from the days of JS pre-promises. If only there was a syntactic solution
that addressed these concerns...

Oh, wait, there is.

```elixir
def add_widgets(widgets) do
    widgets
    |> trim_widget_names()
    |> Enum.uniq_by(fn %{name: val} -> val end)
    |> Enum.filter(fn %{name: val} -> String.length(val) > 4 end)
    |> insert()
end
```

This is not only readable, but easier to reason about and refactor! But, just
how sugary is this syntactic sugar? Surprisingly, not very. The  `|>`  operator
[is a macro](https://github.com/elixir-lang/elixir/blob/v1.10.4/lib/elixir/lib/kernel.ex#L3454),
which maps to Erlang's `foldl/3` (that's how you reference functions in
Erlang/Elixir world - the left side of the slash is the function name, and the
right side is the "arity" - the number of arguments the function accepts).
`foldl/3` accepts a list (which is what the pipe macro creates out of its left
and right hand sides), and invokes them from first to last, passing the
returned value of one in the next.

This is my favorite sort of nicety - no magic, just readability.
[There are some gotchas](https://hexdocs.pm/elixir/Kernel.html#%7C%3E/2) around
parentheses and anonymous functions, but those are more a result of syntactic
ambiguities.

More on a those in a later post.  Happy hacking!
