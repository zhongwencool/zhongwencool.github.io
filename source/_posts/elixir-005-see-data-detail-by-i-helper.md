---
title: iex中查看指定数据详细信息ihelper
date: 2016-04-02 12:41:57
categories: Elixir
tags: [iex]
---
Elixir在1.2后增加了一个新的特性i helper. 在iex shell中使用i可以查看任意数据的数据类型和详细描述
<!-- more -->
![Elixir004_01](http://7fveua.com1.z0.glb.clouddn.com/elixir_004_01.png)

```elixir
#查看变量描述
iex(1)> i {:test, "That sounds great"}
Term
  {:test, "That sounds great"}
  Data type
    Tuple
  Reference modules
    Tuple
#查看Module描述(有点类似于Erlang的lists:module_info)
iex(2)> i List
  Term
    List
  Data type
    Atom
  Module bytecode
    usr/local/Cellar/elixir/1.2.0/bin/../lib/elixir/ebin/Elixir.List.beam
  Source
    private/tmp/elixir20160101-48495-1fg1arr/elixir-1.2.0/lib/elixir/lib/list.ex
  Version
    [322093417650371381585336045669056703278]
  Compile time
    2016-1-1 11:57:45
  Compile options
    [:debug_info]
  Description
    Use h(List) to access its documentation.
    Call List.module_info() to access metadata.
    Raw representation
    :"Elixir.List"
    Reference modules
    Module, Atom
```
来看看这么神奇的功能是怎么实现的吧~
[https://github.com/elixir-lang/elixir/blob/master/lib/iex/lib/iex/helpers.ex#L420](https://github.com/elixir-lang/elixir/blob/master/lib/iex/lib/iex/helpers.ex#L420)
```elixir
@doc """
  Prints information about the given data type.
    """
  def i(term) do
    info = ["Term": inspect(term)] ++ IEx.Info.info(term)
          
    for {subject, info} <- info do
      info = info |> to_string() |> String.strip() |> String.replace("\n", "\n  ")
      IO.puts IEx.color(:eval_result, to_string(subject))
      IO.puts IEx.color(:eval_info, "  #{info}")
    end
                                    
    dont_display_result
  end
```
可以看出它只是把IEx.info.info的结果打出来，我们再看看它发生了什么？

[https://github.com/elixir-lang/elixir/blob/master/lib/iex/lib/iex/info.ex](https://github.com/elixir-lang/elixir/blob/master/lib/iex/lib/iex/info.ex)
```elixir
defprotocol IEx.Info do
  @fallback_to_any true
  
  @spec info(term) :: [{atom, String.t}]
  def info(term)
end
      
defimpl IEx.Info, for: Tuple do
  def info(_tuple) do
    ["Data type": "Tuple",
    "Reference modules": "Tuple"]
  end
end
```
它是一个protocol，在这个文件中把elixir的基本类型都实现了一次，它会返回一个keyword list, 所以我们才能看到，那么如果我们试试自己定义？

```elixir
iex(3)> defmodule User do
…(3)> defstruct name: "John", age: 25
…(3)> @type t :: %User{name: String.t, age: non_neg_integer}
…(3)> end
```
接下来， 我们来自定义看看
```elixir
iex(5)> defimpl IEx.Info, for: User do
…(5)> def info(item) do
…(5)>   ["Data type":  User, "Description": "The customer is god, pleasure they", "Reference": "blablabla..."]
…(5)> end
…(5)> end
iex(6)> i %User{}
Term
  %User{age: 25, name: "John"}
  Data type
    Elixir.User
  Description
    The customer is god, pleasure they
  Reference
    blablabla...
```
成功！

官方文档：[http://elixir-lang.org/docs/stable/iex/IEx.Helpers.html#i/1](http://elixir-lang.org/docs/stable/iex/IEx.Helpers.html#i/1)

彩蛋：

有没有看到我们输入i得到的结果，只是把格式用有颜色的格式打印出来，但是确没有看到返回值被打印出来。。。

它的结果无论如何都打印不出来滴。因为它调用了
```elixir
IEx.dont_display_result
```
在 [evaluator.ex](https://github.com/elixir-lang/elixir/blob/b6dc389f5e3eb2286885c172009333ef414d7860/lib/iex/lib/iex/evaluator.ex#L136) 里面：
```elixir
unless result == IEx.dont_display_result, do: io_inspect(result)
```

