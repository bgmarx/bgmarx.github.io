---
layout: post
title: Sending and Receiving Packets with UDP
tags:
- Elixir
- Erlang
- udp
---



Let's take a look at how to use UDP with Elixir. _User Datagram Protocol_, or [UDP](https://en.wikipedia.org/wiki/User_Datagram_Protocol), like TCP, is one of the core internet protocols. It was designed in 1980 by David Reed. 

UDP is a connectionless communication model and, as a result, doesn't require any previous communication. All you need is an open socket, and you can start sending packets---datagrams---to other hosts. 

Unlike TCP, there is no handshake, and there are no guarantees around delivery, order, or duplicate packets sent. If you're using UDP, you must be willing to accept some packet loss. The tradeoff, as you might imagine,  is that UDP faster and more lightweight. You give up the delivery and ordering guarantees for speed. If you need those guarantees, TCP is the way to go.

Since UDP is connectionless, one can broadcast packets to all devices on some network. Continuing with the connectionless theme, UDP can also multicast. In the context of UDP, multicast is to send a single packet to any number of subscribers. 

All of these qualities make UDP a good bet for streaming, VOIP, or gaming applications where the primary concern is speed, and some packet loss is acceptable.

With a cursory understanding of UDP, let's move on to how to send and receive UDP packets with Elixir.

### Sending and Receiving Packets with :gen_udp 


As everyone knows, Elixir is built on top of Erlang, which means two things generally: using the BEAM virtual machine and having access to any of Erlang's functions and modules. This was---and continues to be---a great advantage for Elixir. In the early days of Elixir, developers already had access to all the Erlang packages and lower-level modules like `:gen_udp`. [`:gen_udp`](http://erlang.org/doc/man/gen_udp.html) is, in the Erlang tradition of `gen_*`, a generic interface for all things UDP. It's a terse module with only a few exported functions befitting UDP.

Let's open a socket, send a message, and verify that the receiver received the message. We can do all this from two iex sessions. Open two terminal windows and start `iex` in each window. 

As you can see from the erlang docs, `open/1` in Erlang parlance `open(Port) -> {ok, Socket} | {error, Reason}` takes a port and returns either an `{:ok, socket}` tuple or an `{:error, reason}` tuple. Pick a port, let's says `8679` and hope the socket opens.

```elixir 
Interactive Elixir (1.9.1) - press Ctrl+C to exit (type h() ENTER for help)
iex(1)> {:ok, socket} = :gen_udp.open(8679)
{:ok, #Port<0.5>}
```

Look at that---it worked. 

Now in the other `iex` session, let's send a message. We can do that with `send/4` or again in Erlang parlance `send(Socket, Host, Port, Packet) -> ok | {error, Reason}`. We need to open a distinct `socket`, provide a host---`{127,0,0,1}`, a port---`8679` and a `Packet` or message. We'll look at the composition of an actual UDP packet later. For now, it's simply a message. Like we did in the first window, we need to open a socket---a different one, to be clear.

```elixir 
Interactive Elixir (1.9.1) - press Ctrl+C to exit (type h() ENTER for help)
iex(1)> {:ok, socket} = :gen_udp.open(8680)
{:ok, #Port<0.5>}
```

With the port open, we can `send/4` a message to port `8679`. It's still open and waiting. 

```elixir 
iex(2)> :gen_udp.send(socket, {127,0,0,1}, 8679, "hello")
:ok
```

We pass in the socket, the host, the port, and a message. What better way to begin a conversation than with `hello`. You can see from the `:ok` output that the message was sent. However, in the first terminal window, we see no message. What gives. 

We need to [flush/0](http://erlang.org/doc/man/c.html#flush-0) to see whatever messages might have been sent to it. 

```elixir
iex(2)> flush()
{:udp, #Port<0.5>, {127, 0, 0, 1}, 8680, 'hello'}
:ok
```

There we have it. Send as many messages as you wish and `flush/0` them to see all the messages sent. Notice the 5-tuple beginning with `:udp`. If you're using `:gen_udp` in a GenServer, for instance, you can pattern match with a `handle_info/2` to do something with the message you receive on that socket.

### Passively Recv'ing Packets 

Let's take a look at `recv/2`. From the docs it says "Receives a packet from a socket in passive mode." By default, `open/2` starts in `active` mode. In order to understand the difference between `active` and `passive` mode, we'll need to take a look at [`:inet.setopts/2`](http://erlang.org/doc/man/inet.html#setopts-2). 

When a socket is set to `active` mode, "everything received from the socket is sent as messages to the receiving process." This is probably what you want and it's why it's the default. `passive` must manually receive incoming data using `recv/2`. Note that this `active` and `passive` division also applies to `gen_tcp:recv/2,3` and `gen_sctp:recv/1,2`. 

So let's set up a passive connection and show how that differs from an active connection. Like before, start two `iex` sessions and in the first one, type the following:

```elixir 
iex(1)> {:ok, socket} = :gen_udp.open(8679, [:binary, {:active, false}])
{:ok, #Port<0.5>}
```

In the other session, start an active session as before:

```elixir 
iex(1)> {:ok, socket} = :gen_udp.open(8680)
{:ok, #Port<0.5>}
```

Now, send a message to port `8679`. 

```elixir 
iex(2)> :gen_udp.send(socket, {127,0,0,1}, 8679, "hello")
:ok
```

Let's `flush/0` to see if the message arrived. 

```elixir
iex(2)> flush()
:ok
```

As expected, no message has arrived. Let's use `recv/2` to see if the message will arrive. The first argument is the `socket` and the second is the length of the packet. In the docs it says the packet can be `Length = integer() >= 0`. 

```elixir
iex(3)> :gen_udp.recv(socket, 0)
{:ok, { {127, 0, 0, 1}, 8680, "hello"} }

```

It arrives as soon as the socket is set up to receive messages. Curiously, the message still arrives even though the actual message size must be bigger than zero. 

You might have noticed that there's a `recv/3`, as well. Most functions in Erlang and Elixir that wait on something have an option `timeout` parameter. It makes sense; there's no real guarantee that something might arrive from somewhere else. So do you wait endlessly or fail and do something else? I generally set pretty aggressively---but within reason---low timeouts. I'd rather find out sooner than later. And so, if you run `recv/2` there's an implied timeout of `infinity`. If you want to see how `recv/3` operates, add an integer in milliseconds as the third argument. 

```elixir
iex(5)> :gen_udp.recv(socket, 0, 5_000)
{:error, :timeout}
```

Again, in the tried and true Erlang fashion, there's an `:error` tuple you can match on and respond accordingly. 
