---
layout: post
title: Binary Pattern Matching with Elixir
tags:
- Elixir
- binary
- Erlang
- pattern matching
---

Pattern matching is one of the more elegant aspects of Elixir (and Erlang). It makes for more readable, terse code. For instance, look at the following function heads.

```elixir
defmodule Options do
  def parse(input) when is_binary(input), do: :binary
  def parse(input) when is_integer(input), do: :integer
  def parse(input) when is_list(input), do: :list
  def parse(_input), do: :error
end
```

You can try with the different types and it works as expected.

```elixir
iex(9)> Options.parse "a"                                
:binary
iex(10)> Options.parse 1  
:integer
iex(11)> Options.parse []
:list
iex(12)> Options.parse :a
:error
```

This is all pretty straight-forward and something that most Elixir users are familiar with. Let's take a look at how to pattern match binaries in Elixir. First, let's take a detour and talk about how Elixir represents strings.

### Strings are Binaries 

Every string is UTF-8 encoded, and in Elixir, binaries are denoted by `<<` and `>>`. For instance, `<<"a">` is a binary which also happens to be a string.

```elixir
iex(2)> String.valid? <<"a">>
true
```

To get the size of a binary, use `byte_size/1`. Each integer is eight bits or one byte.

```elixir
iex(3)> byte_size <<1>>
1
```

Since strings in Elixir are UTF-8, some character sizes are greater than one byte. For example, the Hebrew letter _ח_ or the Japanese Kanji for dog _犬 (inu)_.

```elixir
iex(4)> byte_size <<"ח">>
2
```

```elixir
iex(5)> byte_size <<"犬">> 
3
```

The Hebrew character _ח_ is 2 bytes whereas the Japanese Kanji character _犬_ is 3 bytes. 

By the way, since Elixir strings are UTF-8 both characters are valid strings.

```elixir
iex(6)> String.valid? "ח"
true
```

```elixir
iex(7)> String.valid? "犬"
true
```

### Pattern Matching String Binaries


Now that we have a basic understanding of Elixir strings, let's try pattern matching on a binary. We can pattern match simply an 8 byte binary by assigning it to some variable like the following. 

```elixir
iex(36)> <<a>> = <<15>>
<<15>>
iex(37)> a
15
```

An integer is 1 byte it matches.  You can be explicit with the syntax `<<var :: bit_size>>`.


```elixir
iex(42)> <<a :: 8 >> = <<15>>
<<15>>
```

Now we know that the Hebrew character is 16 bits and the Japanese character is 24 bits. If we try to pattern match a 16---or any non-8 bit---binary against an 8-bit binary, we'll get a pattern match error.

```elixir
iex(43)> <<a>> = <<"ח">>     
** (MatchError) no match of right hand side value: "ח"
    (stdlib) erl_eval.erl:453: :erl_eval.expr/5
    (iex) lib/iex/evaluator.ex:257: IEx.Evaluator.handle_eval/5
    (iex) lib/iex/evaluator.ex:237: IEx.Evaluator.do_eval/3
    (iex) lib/iex/evaluator.ex:215: IEx.Evaluator.eval/3
    (iex) lib/iex/evaluator.ex:103: IEx.Evaluator.loop/1
    (iex) lib/iex/evaluator.ex:27: IEx.Evaluator.init/4
```

If we correctly match by explicitly setting the expected number of bits, everything works as expected. 

```elixir
iex(95)> <<a :: 16 >> = <<"ח">> 
"ח"
```

But, if you see what `a` is assigned to, it might be a surprise.

```elixir
iex(98)> a
55191
```

From the [Special Forms docs](https://hexdocs.pm/elixir/master/Kernel.SpecialForms.html#%3C%3C%3E%3E/1-types), "When no type is specified, the default is `integer`". Let's try again with `bitstring-size(n)`.

```elixir
iex(95)> <<a :: bitstring-size(16) >> = <<"ח">> 
"ח"
iex(96)> a
"ח"
```

```elixir
iex(92)> <<a :: bitstring-size(24) >> = <<"犬">>
"犬"
iex(93)> a
"犬"
```

Everything works as expected.


### Pattern Matching Binaries

Let's try matching on something a bit more involved. As noted above, you can specify the size you expect the binary to be. You can also assign variables to each chunk you want to capture.

```elixir
iex(50)> << a :: size(8), b :: 8, c :: 16 >> = <<1,2,3,4>>
<<1, 2, 3, 4>>
iex(51)> a
1
iex(52)> b
2
```

Or, if you only care about the first eight bytes, you can do the following.


```elixir
iex(58)>  << a :: 8, _rest :: binary >> = <<1,2,3,4>>
<<1, 2, 3, 4>>
iex(59)> a
1
```

This makes it fairly straight-forward to match no more complex binaries. Imagine you have a binary that's composed of a header of 1 byte, a length of 4 bytes, and a message that's 18 bytes.

```elixir
iex(61)> msg_header = <<1>>
<<1>>
iex(62)> msg_length = <<2,3,4,5>>
<<2, 3, 4, 5>>
iex(63)> msg_body = <<"erase the evidence">>
"erase the evidence"
iex(65)> msg_header <> msg_length <> msg_body
<<1, 41, 101, 114, 97, 115, 101, 32, 116, 104, 101, 32, 101, 118, 105, 100, 101,
  110, 99, 101>>
```
Our binary is composed, so let's pattern match on the elements.

```elixir
iex(114)> << header :: 8, length :: 32, message :: bitstring-size(144) >> = msg_header <> msg_length <> msg_body           
<<1, 2, 3, 4, 5, 101, 114, 97, 115, 101, 32, 116, 104, 101, 32, 101, 118, 105,
  100, 101, 110, 99, 101>>
iex(115)> header
1
iex(116)> length
33752069
iex(117)> message
"erase the evidence"
```

That's all there is to pattern matching with binaries in Elixir.