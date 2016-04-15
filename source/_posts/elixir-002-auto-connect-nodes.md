---
title: 节点启动后自动连接其它配置节点
date: 2016-03-02 00:02:44
categories: Elixir
tags: node
---
问题： 如何指定一个节点在启动后自动连接到别的节点上？
<!-- more -->
这个我们要使用到[sys.config](http://erlang.org/doc/man/config.html),这是erlang的配置文件，这个文件一般都是$ROOT/releases/Vsn下

## 首先我们要先启动一个master节点,Node.list可以看到当前节点并没有连接到任何节点
```elixir
iex --cookie secret --name master@127.0.0.1
Erlang/OTP 18 [erts-7.2.1] [source] [64-bit] [smp:4:4] [async-threads:10] [hipe] [kernel-poll:false] [dtrace]

Interactive Elixir (1.2.2) - press Ctrl+C to exit (type h() ENTER for help)
iex(master@127.0.0.1)1>Node.list
[]
```
## 我们再启动一个slave节点，让它启动时直接连接到master上

新建sys.config

```elixir
#sys.config
[{kernel,
  [
   {sync_nodes_optional, ['master@127.0.0.1']},
   {sync_nodes_timeout, 150000}
  ]}
].
```

这个配置可以表明，启动后会主动连接到'master@127.0.0.1' ，如果没有连接成功会每150秒再重连。

```elixir
iex --cookie secret --name slave@127.0.0.1 --erl "-config sys.config"
Erlang/OTP 18 [erts-7.2.1] [source] [64-bit] [smp:4:4] [async-threads:10] [hipe] [kernel-poll:false] [dtrace]

Interactive Elixir (1.2.2) - press Ctrl+C to exit (type h() ENTER for help)
iex(slave@127.0.0.1)1> Node.list
[:"master@127.0.0.1"]
```
这时在2个节点上用Node.list都可以看到对方啦。 注意： 一定要保持2个节点的cookie是一样的

再验证一下：

```elixir
iex(slave@127.0.0.1)2>:rpc.call(:"master@127.0.0.1", :io, :format, [:user, "~ts pid: ~p~n", ["我是在master节点上运行的", self()]])
:ok
iex(slave@127.0.0.1)2>self
#PID<0.69.0>
```
这个format只是会在原运行节点上输出，因为使用了:user， 所以我们在master上看到结果
```elixir
iex(master@127.0.0.1)2> 我是在master节点上运行的 pid: <9547.69.0>
```
可以看到master上输入的pid是slaver上的进程pid, 其中9547是标记它来自slaver@127.0.0.1节点, 即： <0.69.0> == <9547.0.0>

## sys.config实现了发布新的release后会根据新的sys.config再加载配置

>When traversing the contents of sys.config and a filename is encountered, its contents are read and merged with the result so far. When an application configuration tuple{Application, Env} is found, it is merged with the result so far. Merging means that new parameters are added and existing parameter values overwritten.

官方的例子给得太简单啦，演示个完整的例子看看：）

假如我们想在上面的slaver节点上的kernel再加一个配置database的地址， 那么我们新建一个region.config 

```elixir
#File: /etc/region.config
[{kernel, [
          {database_for_example_purposes, "db://prod-host-omg:8089"}
          ]}].
```
那么我们要做的就是把region.config加到sys.config

```elixir
#File: sys.config
[{kernel, [
          {sync_nodes_optional, ['master@127.0.0.1']},
          {sync_nodes_timeout, 150000}
          ]},
          "/etc/region.config"].
```
我们再验证一下是否改变

```elixir
iex --cookie secret --name slave@127.0.0.1 --erl "-config ./sys.config"
Erlang/OTP 18 [erts-7.2.1] [source] [64-bit] [smp:4:4] [async-threads:10] [hipe] [kernel-poll:false] [dtrace]

Interactive Elixir (1.2.2) - press Ctrl+C to exit (type h() ENTER for help)
iex(slave@127.0.0.1)1> Application.get_all_env(:kernel)
[database_for_example_purposes: 'db://prod-host-omg:8089',
 sync_nodes_timeout: 150000, included_applications: [],
 sync_nodes_optional: [:"master@127.0.0.1"], error_logger: :tty]
```
有兴趣可以看看extrm release[如何生成sys.config的](https://github.com/bitwalker/exrm/blob/bebc97c7707b6019a2790132b16653418f25afdc/lib/exrm/utils.ex#L164)

参照资料

1. [Distributed-otp-applications](http://learnyousomeerlang.com/distributed-otp-applications)

2. [Design_principles about distributed_applications](http://erlang.org/doc/design_principles/distributed_applications.html)

3. [sys.config](http://erlang.org/doc/man/config.html)

 





