---
title: 使用behaviour来规范模块对外接口
date: 2016-04-06 23:20:54
categories: Elixir
tags: [GenServer]
---
Erlang/Elixir的Behaviour类似于其它语言中的接口(interfaces),本质就是在指定behaviours的模块中强制要求导出一些指定的函数,否则编译时会warning。
<!-- more -->

其中Elixir中使用到behaviour的典范就是[GenServer](https://github.com/elixir-lang/elixir/blob/master/lib/elixir/lib/gen_server.ex), [GenEvent](http://elixir-lang.org/docs/stable/elixir/GenEvent.html)。

曾经Elixir有一个叫Behaviour的模块，但是在1.1时就已被deprecated掉了，现在你并不需要用一个Behaviour模块才能定义一个behaviour啦。

让我们一步步实现一个自定义的behaviour吧。

## Warehouse和Warehouse.Apple测试用例
我们来定义一个仓库，它只要进货和出货，它的状态就是库存量和类型，然后再在上一层封装一个具体的apple仓库，它基于仓库，但它对外只显示库存量。
```elixir
mix new behaviour_play
cd behaviour_play
```
先写测试，搞明白我们希望的效果
```elixir
# test/behaviour_play_test.exs
defmodule BehaviourPlayTest do
  use ExUnit.Case
  doctest BehaviourPlay
    
  test "warehouse working" do
    :ok = Warehouse.new(%{category: :fruit, store: 0})
    assert Warehouse.query(:fruit) == %{category: :fruit, store: 0}
    :ok = Warehouse.increase(:fruit, 10)
    assert Warehouse.query(:fruit) == %{category: :fruit, store: 10}
    :ok = Warehouse.decrease(:fruit, 2)
    assert Warehouse.query(:fruit) == %{category: :fruit, store: 8}
    assert {:not_enough, 8} = Warehouse.decrease(:fruit, 9)
  end
  # 隐藏了是什么类型的水果这个参数
  test "add a apple warehouse" do
    :ok = Warehouse.Apple.new
    assert Warehouse.Apple.query == 0
    :ok = Warehouse.Apple.increase(5)
    assert Warehouse.Apple.query == 5
    assert {:not_enough, 5} == Warehouse.Apple.decrease(6)
    :ok = Warehouse.Apple.decrease(4)
    assert Warehouse.Apple.query == 1
  end
end
```
我们现在运行测试肯定是失败的，因为我们还没写warehouse.ex，但是先**写测试**是一个好习惯~
## 构建Warehouse和Warehouse.Apple
```elixir
# lib/warehouse.ex
defmodule Warehouse do  
  use GenServer
  def new(%{category: category, store: store} = init_state) when is_integer(store) and store >= 0 do
    {:ok, _pid} = GenServer.start(__MODULE__, init_state, name: category)
    :ok
  end
  def query(pid) do
    GenServer.call(pid, :query)
  end
  def increase(pid, num) when is_integer(num) and num > 0 do
    GenServer.call(pid, {:increase, num})
  end
  def decrease(pid, num)when is_integer(num) and num > 0 do
    GenServer.call(pid, {:decrease, num})
  end
  # GenServer callback
  def handle_call(:query, _from, state) do
    {:reply, state, state}
  end
  def handle_call({:increase, num}, _from, state) do
    {:reply, :ok, %{state| store: state.store + num}}
  end
  def handle_call({:decrease, num}, _from, state = %{store: store})when store >= num do
    {:reply, :ok, %{state| store: store - num}}
  end
  def handle_call({:decrease, _num}, _from, state) do
    {:reply, {:not_enough, state.store}, state}
  end
end
```
以上我们为把每一个warehouse都定义成新建立的一个GenServer进程。这时我们运行一下测试(mix test)，会发现测试1已通过，但是我们具体指定到某一种类型apple的仓库还没有建出来。
```elixir
# lib/apple.ex
defmodule Warehouse.Apple do
  def new do
    Warehouse.new(%{category: __MODULE__, store: 0})
  end
  def query do
    state = Warehouse.query(__MODULE__)
    state.store
  end
  def increase(num) do
    Warehouse.increase(__MODULE__, num)
  end
end
```
上面我们故意少定义了**decrease/1**这个函数，但是执行mix compile， 我们居然可以无任何warning的通过啦，我们只有到运行test时才能发现这个错。

可是希望的结果是在**编译期间就能检查出这种低级失误**来，而不是要到使用时才发现(如果我们没有完备的test，就发现不了啦)

所以我们加入behaviour。
## 使用behaviour包装Apple
```elixir
# lib/warehouse.ex的最前面加上@callback属性
defmodule Warehouse do
  @callback new() :: :ok
  @callback query() :: number
  @callback increase(num :: number) :: :ok
  @callback decrease(num :: number) :: :ok| {:not_enough, number}
  #接原来的内容...
```
然后在apple仓库中引入这个behaviour

```elixir
# lib/apple.ex
defmodule Warehouse.Apple do
  @behaviour Warehouse
  # 接原来的内容
```
这时再compile就可以看到对应的warning啦。
```elixir
> mix compile
Compiled lib/warehouse.ex
lib/apple.ex:1: warning: undefined behaviour function decrease/1 (for behaviour Warehouse)
```
我们再把按指示把decrease/1补全就
```elixir
# lib/apple.ex
def decrease(num) do
  Warehouse.decrease(__MODULE__, num)
end
```
mix compile & mix test就全过啦。
## 几个小细节

###  use GenServer后的callback并没有doc,不会显示在help里面
我们use GenServer后，会自动生成GenServer对应的6个callback函数。但是当我们使用
```elixir
iex(1)> h Warehouse. #按下tab自动补全
Apple  decrease/2    increase/2    new/1   query/1
```
并没有看到这些callback函数的doc...
### use GenServer后为什么可以不必定义全部的callback.
可以在这里看看use GenServer时发生了什么，它先使用**@behaviour** [GenServer](https://github.com/elixir-lang/elixir/blob/master/lib/elixir/lib/gen_server.ex#L415), 希望定义这6个函数的缺省行为，最后再把他们[defoverridable](https://github.com/elixir-lang/elixir/blob/master/lib/elixir/lib/gen_server.ex#L459)。

这就是为什么我们在Warehouse里面没有定义init/1时它却没有warning的原因。这如果在Erlang中就是会warning的，因为他们没有这么flexible 的 Macro系统。
### use实现的原理
可以参照看[elixir的源代码](https://github.com/elixir-lang/elixir/blob/v0.14.0/lib/elixir/lib/kernel.ex#L3531-L3532)， 你需要在原模块定义一个宏__using__，所以我们的终极版本应该是
```elixir
# lib/warehouse.ex 添加
defmacro __using__(options) do
  quote location: :keep do # 如果出错，把错误的error trace打到本模块来
    @behaviour Warehouse
    def new, do: Warehouse.new(%{category: unquote(options)[:category], store: 0})
    def query, do: Warehouse.query(unquote(options)[:category]).store
    def increase(num), do: Warehouse.increase(unquote(options)[:category], num)
    def decrease(num), do: Warehouse.decrease(unquote(options)[:category], num)
    defoverridable [new: 0, query: 0, increase: 1, decrease: 1]
  end
end
```
然后把apple.ex里面只需要use Warehouse一句话就搞定啦。

所以我们可以如果想定义很多的水果apple, banana,  pear，基本就是一句话的事~
```elixir
#lib/apple.ex
defmodule Warehouse.Apple do
  use Warehouse, category: __MODULE__
end
#lib/banana.ex
defmodule Warehouse.Banana do
  use Warehouse, category: __MODULE__
end
# lib/pear.ex
defmodule Warehouse.Pear do
  use Warehouse, category: __MODULE__
end
```
这样的封装就很简化了很多代码，是不是感觉写elixir很爽呀~~~


