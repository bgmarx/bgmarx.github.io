---
layout: post
title: Dirty Schedulers
tags:
- Elixir
- Schedulers
- Dirty Scheduler
- NIF
---

Dirty schedulers officially arrived in OTP 21 and for me they're one of the most exciting features to land in OTP 21. In this post, we'll cover what dirty schedulers are, why they came about and how they make interoperability with NIFs safer and open to more applications.

##### How Schedulers Work

To understand why dirty schedulers behave the way they do and their reason for existing, we first need to discuss how schedulers work on the BEAM.

##### Dirty Schedulers
> “Due to heroic efforts by Steve Vinoski, Rickard Green, and Sverker Eriksson, we have an (experimental) so-called dirty-scheduler (DS) NIF API in the Erlang runtime, which has been somewhat stable since release 17.3.”
>  https://medium.com/@jlouis666/erlang-dirty-scheduler-overhead-6e1219dcc7
