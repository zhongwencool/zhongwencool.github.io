---
title: NestedModule的动态函数调用方式
date: 2016-04-05 23:59:29
categories: Elixir
tags: [module]
---
有时我们需要动态生成一些模块名，然后调用它里面的函数。但是我们常常碰到的却是明明有那个模块，结果还是raise模块未定义。。。
<!-- more -->
我们来看看到底怎么回事？

首先我们定义一个函数
```elixir
iex(1)> defmodule Science.Math do
...(1)> def add(x,y) do
...(1)> x + y
...(1)> end
...(1)> end
```
当我们正常调用它是，是没有问题的：
```elixir
iex(2)> Science.Math.add 1,2
3
```
但是
```elixir
iex(3)> t= "Science.Math"
"Science.Math"
iex(4)> :"Science.Math".add(1, 2)
** (UndefinedFunctionError) undefined function :"Science.Math".add/2 (module :"Science.Math" is not available)
    :"Science.Math".add(1, 2)
```
为什么明明有的模块却没找到呢？

因为我们定义的模块其实应该是:"Elixir.Science.Math", 而不是:"Science.Math"

所以我们应该使用
```elixir
iex(5)> (t|> String.split(".") |> Module.concat).add(1,2)
3
```
但是Module做String to Atom的转换时用的是[:erlang.binary_to_atom(string, :utf8)](https://github.com/elixir-lang/elixir/blob/master/lib/elixir/src/elixir_aliases.erl#L107)

如果在运行期间大量调用这个函数可能造成原子数耗尽(erlang的atom是不回收的)

所以我们还可以使用**Module.safe_concat/1**
```elixir
iex(5)> (t|> String.split(".") |> Module.safe_concat).add(1,2)
3
```
safe_concat使用的 :erlang.binary_to_existing_atom(string, :utf8).

这解决了上面这个函数的风险，但是也引入别一个可能，有可能输入的string是一个系统从来没有exist的atom就会crash掉。

如果我们只是想调用**Nested.Foo.Test**模块里面的函数， 还可以再用一种巧妙的方式。
```elixir
iex(6)> t = "Science.Math"
"Science.Math"
iex(7)> :"Elixir.#{t}".add(1,2)
3
```
最后一种方法只是试探性的，其实你选择的方案应该是最优**Module.safe_concat**,

如果你的需求真的可能输入一个从来没有exist的模块时，你应该使用Module.concat，但是你想一个方法去保证atom不会耗尽。

让我们最后来回顾一下Module最前面的**@moduledoc**
>Provides functions to deal with modules during compilation time.

>It allows a developer to dynamically add, delete and register attributes,
>attach documentation and so forth.

>After a module is compiled, using many of the functions in this module will
>raise errors, since it is out of their scope to inspect runtime data. Most of
>the runtime data can be inspected via the __info__(attr) function attached to
>each compiled module.

它本来就不是用来解决运行期间的问题的。。。。

