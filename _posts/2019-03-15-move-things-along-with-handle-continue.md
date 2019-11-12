---
layout: post
title: Move Things Along with handle_continue
tags:
- Elixir
- Erlang
- GenServer
- handle_continue
- OTP
---

OTP 21 saw the addition of a new callback `handle_continue/2` for GenServer. Let's use an example to show why this new callback was added.

As you probably know, when you start `GenServer.start_link/2` it triggers `GenServer.init/1` callback immediately. `init/1` is a blocking call. Nothing else can happen until `init/1` finishes executing. 

To illustrate that, we'll use a simple example where the argument passed to `start_link/1` is a `timeout` in milliseconds. Of course, a real-world example would be fetching data from a database or waiting on an external resource to load. The timeout example suffices.

```elixir
defmodule SlowInit do
  use GenServer

  def start_link(timeout) do
    GenServer.start_link(__MODULE__, timeout)
  end

  @impl true
  def init(timeout) do
    Process.sleep(timeout)
    {:ok, timeout}
  end
end
```

Open up an `iex` prompt and copy in the code. Then run `SlowInit.start_link(5000)`.

```elixir
{:module, SlowInit,
 <<70, 79, 82, 49, 0, 0, 15, 156, 66, 69, 65, 77, 65, 116, 85, 56, 0, 0, 1, 179,
 0, 0, 0, 43, 19, 69, 108, 105, 120, 105, 114, 46, 65, 112, 112, 46, 83, 108,
 111, 119, 73, 110, 105, 116, 8, 95, 95, ...>>, {:init, 1}}

iex(2)> App.SlowInit.start_link(5000)
# ...5 seconds pass...
{:ok, #PID<0.126.0>}
```

Waiting five seconds---or some other lengthy amount of time---before your GenServer initializes isn't ideal. Before OTP 21, you could do get by with a `send` hack. 

### The Old Way: `send` 

Instead of potentially blocking in the `init/1` callback, you can `send` the GenServer a message and handle the message with a `handle_info/2` callback like so:

```elixir
defmodule SlowInit do
  use GenServer

  def start_link(timeout) do
    GenServer.start_link(__MODULE__, timeout)
  end

  @impl true
  def init(timeout) do
    send(self(), :long_running)
    IO.puts "INIT"
    {:ok, timeout}
  end

  @impl true
  def handle_info(:long_running, timeout) do
    Process.sleep(timeout)
    IO.puts "AFTER TIMEOUT: #{timeout}"
    state = timeout
    {:noreply, state}
  end
end
```

The blocking call has been moved out of `init/1`, which means that `init/1` initializes right away, and the GenServer can start accepting requests. Copy the GenServer code above into an `iex` prompt to verify.

```elixir 
{:module, SlowInit,
 <<70, 79, 82, 49, 0, 0, 15, 192, 66, 69, 65, 77, 65, 116, 85, 56, 0, 0, 1, 215,
 0, 0, 0, 47, 15, 69, 108, 105, 120, 105, 114, 46, 83, 108, 111, 119, 73, 110,
 105, 116, 8, 95, 95, 105, 110, 102, 111, ...>>, {:handle_info, 2}}

iex(4)> SlowInit.start_link(5000)
INIT
{:ok, #PID<0.160.0>}

iex(5)>

AFTER TIMEOUT: 5000
```

As you can see, `init/1` immediately `send`'s the message to the `handle_info/2` callback, prints out `INIT`, and then the `{:ok, pid}` tuple signaling the initialization of the GenServer. The `iex` prompt increments and you can start entering commands, and after the five second timeout completes, `AFTER TIMEOUT: 5000` gets printed to the console. 

### What's bad about the `send` hack?

That works well enough for this trivial example, but imagine that the slow call was important. It could be anything from a list of banned users to upcoming tour ticket releases, but the fact remains that whatever it is, it's probably crucial to your application. 

Since `init/1` completes right away, there's a window of time before the `handle_info/2` callback finishes, and the `init/1` ends. In that window, a user could call the GenServer and, instead of receiving the expected result, would get an empty result. 

For most use cases, the chances of this happening are pretty slim. Still, if it's business-critical information that must be delivered accurately from the instant the server comes online, that's a problem that you can't guarantee won't happen. 

### The New Way: `handle_continue`

With `handle_continue/2`, the messages are guaranteed to be in order. So, the GenServer initializes right away and starts to receive messages. The messages queue up until the `handle_continue/2` executes, and then the GenServer processes them in the order they were received. 

There's not much to change in our example GenServer to use `handle_continue/2` as the following code illustrates.

```elixir
defmodule SlowInit do
  use GenServer

  def start_link(timeout) do
    GenServer.start_link(__MODULE__, timeout)
  end

  @impl true
  def init(timeout) do
    IO.puts "INIT"
    {:ok, timeout, {:continue, :long_running}}
  end

  @impl true
  def handle_continue(:long_running, timeout) do
    Process.sleep(timeout)
    IO.puts "AFTER TIMEOUT: #{timeout}"
    state = timeout
    {:noreply, state}
  end
end
```

The `init/1` changes its return to a tuple of the form `{:ok, state, {:handle_continue, :atom_callback}}`. In this example, it's `{:ok, timeout, {:handle_continue, :long_running}}`. The `send/2` is removed and the `handle_info/2` becomes `handle_continue/2`. It's a small change, and the resulting code is clearer and evokes its intent.

Another advantage of `handle_continue/2` is that you can use it to manage state changes in GenServer initialization. 

If you're on at least OTP 21, it's a small change to take advantage of `handle_continue/2`. 