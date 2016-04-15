---
title: on_definition编译期规范函数定义
date: 2016-04-04 23:25:08
categories: Elixir
tags: [macro]
---
**@on_definition**时会调用**on_def/6**所以我们可以在编译期间对每一个函数自定义你所需要的任何潜规则
<!-- more -->
## 需求
写一个基于memcache的cache模块， 需要在key前面加上特定的前缀, 所以user cache的原始的store函数应该写成
```elixir
# user.ex
def store(user_id, value) do
   key = Cache.key_encode(user_id, :user)
   ...
end
```
由于加前缀的操作**key_encode/1**是所有存入cache前必须要做的事， 所以我们可以考虑通过metaprogramming来定义一个行为叫before_store/2来做这件事,然后在put前hook before_store，但这会让代码非常难以理解。

我觉得更好的方法是在编译store/2期间去检查它的开始有没有执行过这个加前缀的encode函数， 这才能让让代码更容易理解。

所以我们的潜规则是在模块中的每一个函数的第一行，必须是**Cache.key_encode/2**

## on_definition检查模块规则

我们接下来要使用[@on_definition](http://elixir-lang.org/docs/v1.2/elixir/Module.html) 在编译器去检查指定模块是不是符合这个自定义的潜规则。
```elixir
mix new on_definition_play
cd on_definition_play
```

```elixir
# lib/user.ex
defmodule User do
  @on_definition {Cache.Enforcement, :on_def}
  def store_user(user_id, user) do
    key = Cache.key_encode(user_id, :user)
    Cache.put(key, user)
  end
  # 这个是没有做key_encode的例子，应该编译不过
  def store_comment(user_id, comment) do
    Cache.put(user_id, comment)
  end
end
```
看上面的我们定义了on_definition属性，接下来我们就来实现这个**on_def/6**

```elixir
defmodule Cache do
  # 这里只是用到了memcache_client做例子，你可以使用其它backend
  def put(key, value) do
    Memcache.Client.put(key, value)
  end
  def get(key)  do
    Memcache.Client.get(key)
  end
  def key_encode(key, prefix) do
    "#{prefix}:#{inspect key}"
  end
  defmodule Enforcement do
    def on_def(env, _kind, _name, args, _guards, body) do
      check_start_with_key_encode(env, args, body)
    end
    defp check_start_with_key_encode(_env, [{_, meta, _} | _args], body) do
      line = Keyword.get(meta, :line)
      # 从body里面取出第一行，然后再check它的格式 
      expr = get_first_line(body)
      IO.inspect expr
      case expr do
        :print_to_see_this_struct-> # 我们现在也不知道这东西是个什么东西，所以先用IO.inspect/1打出来看看，然后再对格式
          :ok
        _ ->
        raise Cache.LacksEncodeError, message: "Function line#{line} must begin with a Cache.key_encode/2"
      end
    end
    # 定义函数里使用的简略模式 def func,  do: 
    defp get_first_line({:__block__, _, expr_list}) do
      List.first(expr_list)                                                                                                                 
    end
    defp get_first_line(expr)
      expr
    end                                                                                                                                          
  end
  defmodule LacksEncodeError do
    defexception [:message]
  end
end
```
我们也不知道第一行编成AST后会是什么样子，所以我们先把正确的格式给IO.inspect看一看。然后再匹配上去 :)

所以根据inspect的结果我们可以最后把**check_start_with_key_encode/3**写成
```elixir
defp check_start_with_key_encode(_env, [{_, meta, _} | _args], body) do
  line = Keyword.get(meta, :line)
  expr = get_first_line(body)
  case expr do
    {:=, _,
      [{_, _, _},
      {{:., _,
      [{:__aliases__, _, [:Cache]},#就是它！
      :key_encode]}, _,#就是它！
    _}]} ->
      :ok
    _ ->
      raise Cache.LacksEncodeError, message: "Function line#{line} must begin with a Cache.key_encode/2"
  end
end
```
这里再运行mix compile就会得到
```elixir
> mix compile
== Compilation error on file lib/user.ex ==
** (Cache.LacksEncodeError) Function line9 must begin with a Cache.key_encode/2
    lib/cache.ex:31: Cache.Enforcement.check_start_with_key_encode/3
    (stdlib) erl_eval.erl:669: :erl_eval.do_apply/6
```
大功告成！

## 结论：
  **@on_definition**时会调用**on_def/6**所以我们可以在编译期间对每一个函数自定义你所需要的任何潜规则（但是也不要滥用哦:) ）
  
## Resources
  [Module docs](http://elixir-lang.org/docs/v1.2/elixir/Module.html)  这里面还有其它的compile callback函数和选项，值得好好看看

