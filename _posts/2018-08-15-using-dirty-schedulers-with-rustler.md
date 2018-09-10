---
layout: post
title: Using Dirty Schedulers with Rustler
tags:
- Elixir
- Dirty Scheduler
- NIF
- Rust
- Rustler
---

In an earlier post, I showed how to get started using [Rust NIFs with Rustler](/2018/08/01/getting-started-with-rustler/). We'll continue using the same [Nifty repo](http://github.com/bgmarx/nifty). In this post, we'll be using the `dirty_schedulers` branch. Recall that there are two types of dirty schedulers - CPU-bound and IO-bound. In Erlang, these are defined as `ERL_NIF_DIRTY_JOB_CPU_BOUND` and `ERL_NIF_DIRTY_JOB_IO_BOUND`.

In Rustler, they're similarly delimited with an `enum` [here](https://github.com/hansihe/rustler/blob/b6578ea3999fd42f377c2497d8fae0bd629b927d/rustler/src/schedule.rs):

````rust
pub enum SchedulerFlags {
    Normal = 0,
    DirtyCpu = 1,
    DirtyIo = 2,
}
````

As you can see, the names are different, but the idea is the same. A "clean" NIF is a `Normal` NIF, and the dirty scheduler types are `DirtyCpu` and `DirtyIo`.  To use the dirty schedulers, bring the `SchedulerFlags` module with `use rustler::schedule::SchedulerFlags`. You can see this in `lib.rs` in the Nifty repo. In [dirty_test.rs](https://github.com/hansihe/rustler/blob/b6578ea3999fd42f377c2497d8fae0bd629b927d/rustler_tests/src/test_dirty.rs), there are two somewhat contrived tests that illustrate both types of dirty schedulers.

````rust
pub fn dirty_cpu<'a>(env: Env<'a>, _: &[Term<'a>]) -> NifResult<Term<'a>> {
    let duration = time::Duration::from_millis(100);
    ::std::thread::sleep(duration);

    Ok(atoms::ok().encode(env))
}

pub fn dirty_io<'a>(env: Env<'a>, _: &[Term<'a>]) -> NifResult<Term<'a>> {
    let duration = time::Duration::from_millis(100);
    ::std::thread::sleep(duration);

    Ok(atoms::ok().encode(env))
}
````

Both of these tests do the same thing - they sleep for 100 milliseconds and return an `:ok` atom. 100 milliseconds is 100 times the 1-millisecond Erlang scheduler preemptive threshold.

I've yet to come up with a good Dirty IO example, but the Dirty CPU example listed above works well enough to get the dirty schedulers to do some work.

There's one thing more to do. Add the function - which we're calling `timed_cpu` to the `rustler_export_nifs!` macro like so:

````rust
rustler_export_nifs! {
    "Elixir.Nifty",
    [("add", 2, add),
    ("sub", 2, sub),
    ("timed_cpu", 0, timed_cpu, SchedulerFlags::DirtyCpu)],
    None
}
````

The striking distinction is that you need to specify  `SchedulerFlags::DirtyCpu`. For completeness, here's the function:
````rust
fn timed_cpu<'a>(env: Env<'a>, _: &[Term<'a>]) -> NifResult<Term<'a>> {
    let duration = time::Duration::from_millis(1_000);
    ::std::thread::sleep(duration);

    Ok(atoms::ok().encode(env))
}
````

It does the exact same thing as the Rustler test except that it sleeps for an entire second - an inordinate amount of time for a scheduler.  Now, in `nifty.ex`, add the following functions:

````elixir
  def timed_cpu(), do: :erlang.nif_error(:nif_not_loaded)

  def spawn_dirty_cpu_and_receive(num_to_spawn \\ 1) do
    range = 1..num_to_spawn
    process = self()
    Enum.map(range, fn (_elem) ->
      spawn_link fn ->
        (send process, {self(), timed_cpu()})
      end
    end)
    |> Enum.map(fn (pid) ->
      receive do { ^pid,  result } -> result end
    end)
  end
````
`timed_cpu/0` functions in the same way as `add/2` and `sub/2`. These functions are passed to the Rust crate and if for whatever reason they're not found a `:nif_not_loaded` Erlang error is returned. The `spawn_dirty_io_and_receive/1` function spawns some number of processes, calls `timed_cpu/0` and waits to receive the message as denoted by the pinned pid.

The sole purpose of this function is to get the dotted lines representing the schedulers on the observer to move. Start the library with `iex -S mix` and from the prompt run `iex(2)> Nifty.spawn_dirty_io(5)`. This spawns five processes which should take a total of five seconds to return with a list of `:ok` atoms.

````elixir
iex(3)> Nifty.spawn_dirty_io_and_receive(5)
[:ok, :ok, :ok, :ok, :ok]
````

Load up the observer and let's see if we can get the dirty schedulers to activate.

<img src="/public/images/dirty-cpu-observer.png" alt="drawing" style="width:100%;"/>

Hey, look at that. It worked. And it used two of the four schedulers.

What's interesting is that if you go back to `lib.rs` and change `SchedulerFlags::DirtyCpu` to `SchedulerFlags::DirtyIo` and run `spawn_dirty_cpu_and_receive/1` again with the observer running, you'll see that none of the dirty schedulers are activated.

<img src="/public/images/dirty-io-observer.png" alt="drawing" style="width:100%;"/>

That makes sense and I'll figure out a way to simulate dirty IO in a future post.
