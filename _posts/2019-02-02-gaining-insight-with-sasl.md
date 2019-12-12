---
layout: post
title: Gaining Insight into an Elixir Application with SASL
tags:
- Elixir
- Erlang
- Logging
- SASL
- alarm handler
---

`:sasl` is an [Erlang application](http://erlang.org/doc/man/sasl_app.html) that can provide insight into how an Erlang or Elixir application is running. In an Elixir application, you can use `:sasl` to format certain reports to Logger to format them in Elixir terms. 

Let's take a look at how to add `:sasl` to an Elixir app, then how `:sasl` provides more information during startup, and why that might be useful.

### Add SASL to an Elixir App

To use SASL in an Elixir app, you need to open `mix.exs` and add it to the `extra_applications` keyword list. Note that `:sasl` _must_ be started before `:logger`.  

```elixir
  def application do
    [
      extra_applications: [:sasl, :logger]
    ]
  end
```

In case you're unaware, `extra_applications` is "a list of OTP applications your application depends on which are not included in `:deps`." Additionally, Mix guarantees that all the applications---dependencies and `extra_applications`---are started before the application itself starts.

It's crucial that applications like `:logger` and `:sasl` start before the application itself does. If the app crashes during start-up, how else would you know? The reason `:sasl` starts before `:logger` is that `:sasl` ["redirects supervisor, crash and progress reports to Logger so they are formatted in Elixir terms."](https://hexdocs.pm/logger/Logger.html)

The last bit we have to do is add `Logger` configuration details. Open up `config.exs` and either add or amend a `:logger` config. All you have to add is `handle_otp_reports: true` and `handle_sasl_reports: true` as the code below illustrates.

```elixir
config :logger,
  format: "$message\n",
  level: String.to_atom(System.get_env("LOG_LEVEL") || "info"),
  handle_otp_reports: true,
  handle_sasl_reports: true
```

`:handle_otp_reports` defaults to `true`, but if you're using OTP 21 or greater, you must explicitly add both `:handle_otp_reports` and `:handle_sasl_reports` in order to see reports formatted in Elixir terms.


### Startup Logs with `:sasl`

Normally, when you start your application with `iex -S mix`, you see that mix loads your app and then the `iex` prompt appears. 

```console
Erlang/OTP 22 [erts-10.4] [source] [64-bit] [smp:12:12] [ds:12:12:10] [async-threads:1] [hipe]

Compiling 1 file (.ex)
Generated sample app
Interactive Elixir (1.9.2) - press Ctrl+C to exit (type h() ENTER for help)
```

But if you enable `:sasl`, you'll see something quite different. In this `sample` app, the only dependency is `:propcheck` as you see below.


```elixir 
 defp deps do
    [
      {:propcheck, "~> 1.1", only: [:test, :dev]}
    ]
  end
```

Again, booting the mix app with SASL reports enabled shows a much more complete story of what's involved when an application boots.

```console
λ ~/ iex -S mix 
Erlang/OTP 22 [erts-10.4] [source] [64-bit] [smp:12:12] [ds:12:12:10] [async-threads:1] [hipe]

Application logger started at :nonode@nohost
Application proper started at :nonode@nohost
Child :dets_sup of Supervisor :kernel_safe_sup started
Pid: #PID<0.188.0>
Start Call: :dets_sup.start_link()
Child :dets of Supervisor :kernel_safe_sup started
Pid: #PID<0.189.0>
Start Call: :dets_server.start_link()
Child PropCheck.CounterStrike of Supervisor Propcheck.Supervisor started
Pid: #PID<0.187.0>
Start Call: PropCheck.CounterStrike.start_link("/Users/user/dev/sample/_build/propcheck.ctex", [name: PropCheck.CounterStrike])
Application propcheck started at :nonode@nohost
Application sample started at :nonode@nohost
Interactive Elixir (1.9.2) - press Ctrl+C to exit (type h() ENTER for help)
iex(1)> 
```

First, you can see that the `sample` application---the mix app itself---is the last application that starts. Moreover, `logger`, `proper`, and its Elixir wrapper `propcheck` are all started. These give credence to the idea that an Elixir application is a collection of other applications---what Lance popularized with [_Phoenix is not your Application_](https://www.youtube.com/watch?v=lDKCSheBc-8).

You can also see that something called `:kernel_safe_sup` is started as well as some other children and a `PropCheck.Supervisor` as well. If you search for `:kernel_safe_sup`, you'll find it in the Erlang [source code](https://github.com/erlang/otp/blob/master/lib/kernel/src/kernel.erl#L55).

```erlang
%%%-----------------------------------------------------------------
%%% The process structure in kernel is as shown in the figure.
%%%
%%%               ---------------
%%%              | kernel_sup (A)|
%%%	          ---------------
%%%                      |
%%%        -------------------------------
%%%       |              |                |
%%%  <std services> -------------   -------------
%%%   (file,code,  | erl_dist (A)| | safe_sup (1)|
%%%    rpc, ...)    -------------   -------------
%%%		          |               |
%%%                  (net_kernel,  (disk_log, pg2,
%%%          	      auth, ...)     ...)
%%%
%%% The rectangular boxes are supervisors.  All supervisors except
%%% for kernel_safe_sup terminates the entire erlang node if any of
%%% their children dies.  Any child that can't be restarted in case
%%% of failure must be placed under one of these supervisors.  Any
%%% other child must be placed under safe_sup.  These children may
%%% be restarted. Be aware that if a child is restarted the old state
%%% and all data will be lost.
%%%-----------------------------------------------------------------
%%% Callback functions for the kernel_sup supervisor.
%%%-----------------------------------------------------------------
```

From the documentation, you can see that "[a]ll supervisors except for kernel_safe_sup terminates the entire erlang node if any of their children dies." While this might seem like trivia, it actually provides a lot of insight into what happens when an Erlang (Elixir) node boots. 

This `sample` app only had one dependency and was built without a supervision tree and application callback (no `--sup` option). If the app were more complex and had more OTP behaviors, `:sasl` would become much more useful.

