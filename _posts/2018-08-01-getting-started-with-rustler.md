---
layout: post
title: Getting Started With Rustler
tags:
- Elixir
- Rust
- NIF
---

Rust is an exciting new language that's advertised "a systems programming language that runs blazingly fast, prevents segfaults, and guarantees thread safety." It excels in the same areas as C except that it has much more robust guarantees regarding memory and type safety.  

When one's Erlang or Elixir program requires speed or number crunching - or both - NIFs (Native(ly) Implemented Functions) are the solution. NIFs are generally written in C  to take advantage of C's speed. The tradeoff with NIFs written in C is that the whole BEAM can crash and won't recover. This is why whenever a NIF is referenced, there's usually a dire warning that a NIF crash will crash the whole VM. With Rust, because of its memory safety and static types, the chance of catastrophic failure is much lower.

Here's the repo to follow along with - [nifty](https://github.com/bgmarx/nifty).

Getting started with Rustler is straightforward. Add `rustler` to `mix.exs`  like any other dependency:

````elixir
  defp deps do
    [
       ...snip...
      {:rustler, "~> 0.18"}
    ]
  end
````

Then, from the command line, run `mix deps.get` and once the dependencies have been fetched, run `mix rustler.new`.

This command sets up both the Rust directory structure and also the interface between Elixir and the Rust NIF. You can follow the output below:

````console
/nifty> mix rustler.new
==> rustler
Compiling 2 files (.erl)
/usr/lib/erlang/lib/parsetools-2.1.2/include/yeccpre.hrl:60: Warning: erlang:get_stacktrace/0: deprecated; use the new try/catch syntax for retrieving the stack backtrace
Compiling 6 files (.ex)
Generated rustler app
==> nifty
This is the name of the Elixir module the NIF module will be registered to.
Module name > Nifty
This is the name used for the generated Rust crate. The default is most likely fine.
Library name (nifty) >
* creating native/nifty/README.md
* creating native/nifty/Cargo.toml
* creating native/nifty/src/lib.rs
Ready to go! See /niftynative/nifty/README.md for further instructions.
````

The generated README.md explains things clearly and succinctly. Let's walk through it. The first thing required is to add the Rust compiler and the just-created crate in `mix.exs`. First, in the `project` function:

````elixir
  def project do
    [
	...snip...
      compilers: [:rustler] ++ Mix.compilers,
      rustler_crates: rustler_crates(),
      ...snip...
    ]
  end
````

Then, create a new private function called `rustler_crates` like so:

````elixir
  defp rustler_crates do
    [nifty: [
      path: "native/nifty",
      mode: (if Mix.env == :prod, do: :release, else: :debug),
    ]]
  end
````

To run the Rust tests alongside the Elixir tests when you run `mix test` add an alias:

````elixir
  defp aliases do
    [
      "test": ["cmd cd native/nifty && cargo test", "test"],
    ]
  end
````
Now, all that's left to do get everything up and running is to `use` Rustler in the Elixir module and add error handling. For simplicity, the Rust function adds two numbers together.  The Elixir function `add/2` raises an error if for whatever reason it can't load the Rust crate.

````elixir
defmodule Nifty do
  use Rustler, otp_app: :nifty, crate: :nifty
  @moduledoc """
  Documentation for Nifty.
  """

  def add(_a, _b), do: :erlang.nif_error(:nif_not_loaded)
end
````

Now, let's turn to the Rust code which lives in `native/nifty/src/lib.rs`.  First, let's take a look at the `add` function.

````rust
fn add<'a>(env: Env<'a>, args: &[Term<'a>]) -> NifResult<Term<'a>> {
    let num1: i64 = try!(args[0].decode());
    let num2: i64 = try!(args[1].decode());

    Ok((atoms::ok(), num1 + num2).encode(env))
}
````

First, the function takes two arguments, the `env` which is passed in by Rustler and a reference to some args and it returns a `NifResult`. The two inputs and the output all have the same lifetime as denoted by `'a`. Lifetimes are the way by which Rust can guarantee memory safety. The compiler guarantees that all references are valid.

The body of the function should be immediately readable, but it says to try to parse the arguments passed in and set them to be type `i64`. Then the two integers are summed and returned as a two-tuple in the form of `{:ok, total}`.

At this point, this should work. It's verifiable by running `iex -S mix` and then from the prompt run `Nifty.add(1,2)`. It'll return `{:ok 3}`.

What happens when you pass non-integers to the function. Let's find out:

````rust
iex(3)> Nifty.add 1, :a
** (ArgumentError) argument error
    (nifty) Nifty.add(1, :a)
````

That's pretty much what one would expect.

To add more functions, create the function and then add it to the `rustler_export_nifs!` macro like so:

````rust
rustler_export_nifs! {
    "Elixir.Nifty",
    [("add", 2, add), ("sub", 2, sub)],
    None
}
````

In this case, there's a function added named `sub` that takes two arguments and is called `sub` in both Rust and Elixir.

And that's all there is to it to getting started with Rustler. In upcoming posts, I'll focus on benchmarking, dirty schedulers and see if we can make Rust cause weird behavior on the BEAM.
