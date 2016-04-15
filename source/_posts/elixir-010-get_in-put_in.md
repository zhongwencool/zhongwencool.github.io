---
title: 辅助处理Deeply Nested Data函数
date: 2016-04-10 22:19:02
categories: Elixir
tags: [macro]
---
[put_in和get_in](http://elixir-lang.org/docs/stable/elixir/Kernel.html#put_in/2)是elixir在很早(0.14)就引入的特性,最开始时叫Kernel.pop_in/2 [changelog](https://github.com/elixir-lang/elixir/blob/master/CHANGELOG.md#v0140-2014-06-08)， 专门用于**方便**处理deeply nested data structures。
<!-- more -->
```elixir
@post %{
    title: "ELixir",
    date: "2016-04-10",
    charge: %{author: [%{name: "zhongwencool"", address: "china"}],
             "proofreader" => [%{name: "robot", address: "earth"}]} ## 故意处理这个key为String的
    }
```
创造上面这个3层nested的数据，一般我们可以使用
```elixir
test "getting data out of a deeply nested structure" do
  assert "zhongwencool" == @post.charge.author.name
  assert "robot" == @post.charge["proofreader"].name
end
```
从上面可以看出使用点(**.**)的访问也是非常直观，但是
1. 如果key不为atom时就不能使用
2. 如果key为atom，但是不存在时会crash掉

基于上面的这些key不同就要使用不同的处理方式，我们都可以通过`get_in/2`, `put_in/3`来解决
## `get_in`

```elixir
test "getting data out of a deeply nested structure when atom key" do
  assert "zhongwencool" == get_in(@post, [:charge, :author, :name])    
end
test "getting data out of a deeplynested structure when string key" do
   assert "robot" == get_in(@post, ["proofreader", :name])
end
test "getting data out of a deeply nested, structure when not exist key" do
  assert nil == get_in(@post, [:no_exit_key])
end
```
`get_in` 虽然没有为我们缩短一丁点的代码(其实还更长了一些),但是它做到了**key not found时不会crash**

## `put_in`

```elixir
test "change author's address" do 
   new_author = %{name: @post.charge.author.name, address: "new_address"}
   new_post %{post | change: %{@post.charge| author: new_author}}
   assert "new_address" == new_post.charge.author.address
end
```
可以看出3层的nested看上去已非常不清晰了
```elixir
test "change author's address by put_in/3" do
   new_post = put_in(@post, [:charge, :author, :address], "new_address")
   assert "new_address" == new_post.charge.author.address
end
test "change author's address by put_in/2"
  new_post = put_in(@post[:charge, :author, :address], "new_address")
  assert "new_address" == new_post.charge.author.address
end
```
一对比，`get_in/3`就会现了在处理nested data structure就显得更加的得心应手啦
但是要注意的时，我们也不能put_in时，nested data中除了最后一个key的其它key必须是存在的，不然会crash
```elixir
iex > put_in(%{}, [:not_exist_key1, :not_exist_key2], 1)
** (ArgumentError) could not put/update key :not_exist_key2 on a nil value
    (elixir) lib/access.ex:191: Access.get_and_update/3
        (elixir) lib/access.ex:182: Access.get_and_update/3
            (elixir) lib/kernel.ex:1721: Kernel.put_in/3
            
iex> put_in(%{test: 1}, [:test, :good], 1)
** (FunctionClauseError) no function clause matching in Access.get_and_update/3
    (elixir) lib/access.ex:169: Access.get_and_update(1, :good, #Function<11.124326177/1 in Kernel.put_in/3>)
        (elixir) lib/access.ex:182: Access.get_and_update/3
            (elixir) lib/kernel.ex:1721: Kernel.put_in/3
            
iex> put_in(%{test: %{}}, [:test, :good], 1)
%{test: %{good: 1}}
```
## `update_in/3`

这个`update_in`只能用于更新data,且最后一个参数一定是`fn/1`
这是为了解决上面这个改变一个值时不需要先把值先取出来，再放回去的步骤。
```elixir
test "change author's address by upate_in/3"
  new_post = update_in(@post, [:charge, :author, :address], fn(address) -> address <> ".shanghai") end)
  assert  "china.shanghai" == new_post.charge.author.address
end
test "change author's address by upate_in/2"
  new_post = update_in(@post[:charge, :author, :address], fn(address) -> address <> ".beijing") end)
  assert  "china.beijing" == new_post.charge.author.address
end
```
## `get_and_update_in`
这个是`update_in`的加强版本，并用function去更新这个值，然后返回更新前值(未改变之前的值)

其实你可以看到`update_in`的实现就是使用`get_and_update_in`来实现的,[源代码](https://github.com/elixir-lang/elixir/blob/master/lib/elixir/lib/kernel.ex#L1759)
```elixir
 test "fetch and update in one" do
   {origin_address, update_address} = get_and_uddate_in(@post, [:charge, :author, :address], &{&1, String.upcase(&1)})
   assert "china" == origin_address
   assert "CHINA" == update_address
 end
 ```
## `Access and @derive and Structs`
因为上面这些macro都是依赖于Access的，所以你如果要在自己定义的struct使用(有木有注意到上面的都是map...),那么就必要告诉它， 这个是什么。
```elixir
defmodule Post do
  defstruct [:title, :date, :charge]
end
```
我们希望使用`get_in`的方式是
```elixir
@post_struct  %Post{
    title: "ELixir",
    date: "2016-04-10",
    charge: %{author: [%{name: "zhongwencool"", address: "china"}],
             "proofreader" => [%{name: "robot", address: "earth"}]}
    }

test "get in a struct" do
   assert "zhongwencool" = get_in(@post_struct, [:charge, :author, :name])
end
```
我们跑不过上面这个test，是因为我们没有实现Access protocol, 所以在定义struct时要指定一下
```elixir
defmodule Post do
  @derive [Access]
  defstruct [:title, :date, :charge]
end
```

## `Resources`
- [@derive docs](http://elixir-lang.org/docs/stable/elixir/Kernel.html#defstruct/1)
- [Clojure's update-in](http://clojuredocs.org/clojure_core/1.2.0/clojure.core/update-in)

