---
title: 使用tty终端做一个简单的日记记录
date: 2016-02-29 22:52:38
categories: Elixir
tags: [tty,io]
---
善用tty布局。
<!-- more -->
大半年前实践的使用Evernote做知识管理 [知乎记录](https://www.zhihu.com/question/20232993/answer/34270710)
里面的记录日记模块大概长成这样子
![test](http://7fveua.com1.z0.glb.clouddn.com/elixir_001_01.jpg)
用了半年，感觉缺点很明显，表格太占空间，每周不会自动生成新的表单，搜索功能弱爆了。容易忘记写。

所以我决定用Elixir的个终端版本的。开始！

## 新建项目
```elixir
mix new lifelog
cd lifelog
```
生成的项目文档如下
```elixir
└── lifelog
├── README.md
├── config
│   └── config.exs
├── lib
│   └── lifelog.ex
├── mix.exs
└── test
├── lifelog_test.exs
└── test_helper.exs
```

## 创建Application
在生成的lifelog.ex中使用Application创建一个工作进程LifeLog.Store，后续会使用它来储存所有的数据，使用重启策略为rest_for_one

```elixir
def start(_type, _args) do
  import Supervisor.Spec
  children =
  [worker(LifeLog.Store, [])]
  Supervisor.start_link(children, strategy: :rest_for_one)
end
```
因为想使用escript的方式来启动Application。mix自带生成escript命令

```elixir
mix escript.build
```
使用这个命令的提示可以

```elixir
mix help escript.build
Example
┃ defmodule MyApp.Mixfile do
┃  def project do
┃    [app: :myapp,
┃    [version: "0.0.1",
┃     escript: escript]
┃  end
┃
┃ def escript do
┃   [main_module: MyApp.CLI]
┃ end
┃ ed
```
全部修改可见 github log: (指定Application启动时进程结构, 并指定生成escript文件方式)[https://github.com/zhongwencool/lifelog/commit/9b798cb469c59040654f1a4f7b71cc29ff15a2df]

要特别注意的是为了让escript支持颜色ANSI显示, 我们在escript中加入了

```elixir
emu_args: "-noinput -elixir ansi_enabled true"
```
那些接下来我们就把escript中指定的入口 LifeLog.CLI完成。

## 完善escript的入口中指定的LifeLog.CLI完成

```elixir
defmodule LifeLog.CLI do
  def main(_args) do
    LifeLog.Watcher.start_link(LifeLog.IO)
    :erlang.hibernate(Kernel, :exit, [:killed])
  end
end
```
这个进程只是新启动一个进程gen_event(watcher 稍后我们再完善它)来监控终端的动作(输入或输出),然后自已就hibernate.

我们再来看看LifeLog.IO.ex， 这个gen_event主要是打开一个端口，并监控此端口的输入，和控制它的输出内容
```elixir
Port.open({:spawn, "tty_sl -c -e"}  # 打开端口)
```
只要打开了端口tty_sl后，那么当 tty终端有输入内容时，此IO的gen_event就会收到消息

```elixir
def handle_info({pid, {:data, data}}, %{pid: pid, buffer: buffer}) do
  case translate(data, buffer) do
    {:continue, buffer} ->
      Store.update(buffer, 0)
    {:ok, %{pid: pid, buffer: buffer}}
      {:done, buffer} ->
        Store.update(buffer, 1)
    {:ok, %{pid: pid, buffer: ""}}
  end
end
```
你每按下一个按键都会发触发这个事件，那个我们需要做的就是

1. 每个按键都告诉Store(这个是用来储存所有数据的),
2. 实现退格  一旦输入del就删除提buffer的最后一字符
3. 输入enter完成, 一旦输入enter就表明本条记录已完成，传给Store后进入下一条

全部修改可见： github log：[cli负责生成script脚本, IO负责开端口使用tty_sl输入和IO.write到所开的端口输出](https://github.com/zhongwencool/lifelog/commit/92f64e29cddc9e3252abc55edd38dcc7e94f4914)

接下来我们再完成一个Store中的储存方式。

## Store的储存逻辑

Store应该为gen_server，他全部责任就是

1. 把数据从文件中读到state中
2. 从IO得到应该update的数据，然后更新 
3. 把数据持久化到文件中

所有修改可见： github log: [所有的数据都存在Store中](https://github.com/zhongwencool/lifelog/commit/af1d063c1667f5691bddb7ea50377fbe453e52be)

我们已经完成了把IO输入的消息都传给Store，

接下来关键在于怎么把Store的数据变化再回写到IO上，让我们能看到！

## 使用Watcher把Store和IO联系起来
在Watcher中把IO进程加到 Store中起到的manager(gen_event)中。
```elixir
%{manager: manager, diary: diary} = LifeLog.Store.peek
:ok = GenEvent.add_mon_handler(manager, LifeLog.IO, diary)
```
这样只要Store对着manager发起notify事件，那么IO进程就会收到！，这样我们就可以把Store上的数据变化再通知到IO上。

所有的修改可见： github log: [Watcher负责把Store(gen_server)和IO(gen_event)连接起来](https://github.com/zhongwencool/lifelog/commit/bb909ba7149070e995d423be40afb079a0891c28)

按下来，我们再看看我们应该用什么格式都数据存在文件中

## 新建文件file.ex 只负责从文件中读数据，和写数据到文件中

1. 可以使用:erlang.term_to_binary/1把map encode to binary写放文件，然后load时使用:erlang.binary_to_term/1来decode文件中的数据
2. 把数据转成json数据格式

为了存数据方便，我们使用了方案2： 通用的json格式存数据，所以引放了鼎鼎大名的poison库来处理json.

全部修改可见： github log: [file负责持久化数据，把数据从文件中load出来](https://github.com/zhongwencool/lifelog/commit/c0bc9eadb77af81df8535946087fa1d006b8d92c)

数据的取存都解决了，那们我们来做一下数据的显示，怎么显示数据到终端。

## 新建formatter.ex 负责界面绘制

不要在IO中加放显示的具体逻辑，让formatter把显示的格式整理好）直接传给IO就行，IO只负责输入和输出,不负责输出的具体逻辑。

所以修改可见： github log: [formatter负责界面绘制](https://github.com/zhongwencool/lifelog/commit/1ff5db3b13f7090d08133334c42582c880cb20fd)

到目前为止我们见到的终端界面是这样子的：

![UI](http://7fveua.com1.z0.glb.clouddn.com/elixir_001_02.jpg)

是不是很弱。。。。。

其实我们的基本功能都实现啦！

接下来就是我们把一周的数据都显示出来啦！

## 显示一周的数据

为了更好的管理一周的数据，我们定义了几个Daily DailyItem WeekItem struct来管理数据结构。

所有修改可见 github log: [新增struct, 使用Posion.decode!/2序列化](https://github.com/zhongwencool/lifelog/commit/16ace9a22371909fec0cf934fab5180a535d02fd)

这里使用了posion可以递归的解析struct的特性，所以使用起来非常好理解。
```elixir
JsonData|>Poison.decode!(as: %Diary{daily_items: [%DailyItem{content: [%Daily{}]}], week_items: [%WeekItem{}]})
```
接下来就是非常繁锁的一点点调界面显示时每个block的大小。。。

这里有一点要注意的是，我们把所有的数据+格式都调整完成后再一次性给IO来做显示的。（而不是一行一行的写)

所有的修改可见： githhub log: [显示界面绘制](https://github.com/zhongwencool/lifelog/commit/680c31edea3d5dbd42cd98c367014c4f7788b9a2) 与 调整布局：[[最后一行统一加下划线](https://github.com/zhongwencool/lifelog/commit/16731c64239383dbf084a318b9a82b7f3ebe89f6)

上面的IO.ANSI也是很好玩的一个lib~

这下子就基本能用啦：）

运行：
```elixir
mix escript.build  && ./lifelog
```
![Elixir003](http://7fveua.com1.z0.glb.clouddn.com/elixir_001_03.jpg)

