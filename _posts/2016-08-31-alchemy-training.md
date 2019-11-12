---
layout: post
title: Alchemy Training
tags:
- Elixir
- Training
- Erlang
- Developers
- Team Buildings
---

This post originally appeared on Bleacher Report's [engineering blog](https://dev.bleacherreport.com) under [Alchemy Training](https://dev.bleacherreport.com/alchemy-training-elixir-at-b-r-59190f3db2a8).

At Bleacher Report, whenever we start development on a new service, the first decision is the language and framework. For the majority of Bleacher Report’s existence that has been Ruby and Rails. But for the past year and a half we’ve invested heavily in Elixir and Phoenix. So much so that every one of our backend developers is now an Elixir developer as well.

People often ask how we’ve been able to quickly and efficiently introduce Elixir and Phoenix to a group of backend developers who were primarily Ruby and Rails developers before.

The first thing we recommend to a developer who wants to learn Elixir and — since we are a web company — Phoenix is to get _Progamming Elixir_ by Dave Thomas and _Progamming Phoenix_ by Chris McCord, Bruce Tate, and Josè Valim. Rather than reading the books cover to cover, we’ve found it better to read only the first sections of _Programming Elixir_ to learn the syntax and to understand some basic FP concepts.

Elixir’s syntax closely resembles Ruby in many ways; this helps lower the barrier to entry because the syntax is immediately recognizable. The developer can then concentrate on learning the functional programming concepts which would be that much more difficult if the language syntax were also unfamiliar. As much as I like Erlang’s syntax, it would have been a much more difficult sell to the development team to learn FP concepts as well as Erlang’s unique syntax. Having a familiar syntax enables developers to quickly start writing Elixir code even if it has Ruby-ish elements to it. That’s OK; over time and through iterations, the developer will be writing idiomatic Elixir and quite quickly at that.

Part of the transition from Ruby and Rails to Elixir and Phoenix has meant rewriting pre-existing services and breaking apart the old monolith. This is usually the first code a new Elixir developer would write. Since the code is already written in familiar Ruby, it’s fairly easy to port that code to Elixir.

In my case, the first Elixir app I wrote was a rewrite of a simple Ruby app. It wasn’t very good Elixir code but it performed better than the Ruby app. That and the ease with which one can write tests in ExUnit gave me a great deal of confidence going forward that I could write Elixir and that I would improve over time. I still didn’t really know what I was doing but I was surprised at how much I enjoyed writing Elixir which helped propel me along. Iterating over that app helped me greatly improve my Elixir code. Coupled with adopting Elixir and Phoenix at such an early stage has encouraged us not only to keep up to date with the latest versions but to emphasize refactoring and iterative development in all of our apps.

We try to follow that with new Elixir developers still; start by porting code from a Ruby app to an Elixir app, get the code working and make sure that the test coverage is high. Functioning code with high test coverage give new developers confidence that not only can they write Elixir code that works but because of the high test coverage, they can refactor without worrying too much about regressions or introducing new bugs.

Once those objectives are met, the code review begins. These code reviews are done in person and help to explain how to write more idiomatic Elixir. Most of the initial code reviews involve rewriting control flow logic to using pattern matching on functions and so on. The revisions are what one would expect from developers moving from OOP to FP.

The BEAM is incredibly efficient and forgiving as well. Even poorly written code — for example my first production Elixir app — performs incredibly well. As we’ve gotten better, the performance gains and server savings have only increased as well. As a pleasant side effect, more idiomatic Elixir code decreases the number of lines of code in the app.

In addition to code reviews from more experienced developers, there are many great libraries now that help maintain code consistency and standards. We use excoveralls to ensure high test coverage. For linting and code standards we use credo. Credo is an indispensable library because not only does it lint the code, it teaches by explaining why the library makes the suggestions it does. Using credo to maintain standards allows for more time to work on business logic and overall application architecture. Both credo and excoveralls are hooked up to our CI ensuring that these standards are maintained and enforced.

Employing these simple guidelines have enabled us to train all of our backend developers (and some frontend devs, too…) to write and maintain Elixir code and Phoenix applications. In fact, most developers who have written or contributed to an Elixir and Phoenix app prefer Elixir and Phoenix to other languages and frameworks. Developer happiness and increased productivity are just one of the many ways our early investment in Elixir and Phoenix has paid off.
