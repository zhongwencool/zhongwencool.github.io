---
title: mix archives build/install完整例子
date: 2016-03-07 23:40
categories: Elixir
tags: mix
---

在[日记记录篇章中](http://zhongwencool.github.io/2016/02/29/Elixir001-%E4%BD%BF%E7%94%A8tty%E5%81%9A%E4%B8%80%E4%B8%AA%E7%AE%80%E5%8D%95%E7%9A%84%E6%97%A5%E8%AE%B0%E8%AE%B0%E5%BD%95/)中使用 mix escript.build 生成一个lifelog 的escript启动脚本。
今天我们尝试一下另一种方式：生成Archives。

<!-- more -->

我们先添加一个[Task](http://elixir-lang.org/docs/stable/mix/Mix.Task.html)

## 查看下我们现在有那一些task

这个命令非常有用，:)
```elixir
> mix help
mix                   # Runs the default task (current: "mix run")
mix app.start         # Starts all registered apps
mix archive           # Lists all archives
mix archive.build     # Archives this project into a .ez file
mix archive.install   # Installs an archive locally
mix archive.uninstall # Uninstalls archives
mix clean             # Deletes generated application files
.....
```
## 创建项目自定义的Task

```elixir
> mkdir -p lib/mix/tasks  #一般都是放在lib/mix/tasks下的！
>emacs lib/mix/tasks/lifelog.start.ex # 一般都是以.分隔的！
defmodule Mix.Tasks.Lifelog.Start do
  use Mix.Task
  @shortdoc "Start logging"
  @moduledoc """
    Manager your life.
  """
  def run(args) do
    { _opts, args, _ } = OptionParser.parse(args) # 把命令行的args parse成可以使用的args
    Mix.Tasks.App.Start.run(args) #要先把application先启动
    LifeLog.CLI.main(args)
  end
end
```
现在我们可以在当前目录下运行
```elixir
> mix lifelog.start  #在项目当前目录下
```
成功打开录入界面!!!!!!! ，但是如果在别的目录下呢？
## 构建archive
现在我们如果不在项目的root
```elixir
> mix lifelog.start
** (Mix) The task "lifelog.start" could not be found
```
根本找不到我们的这个task，因为我们还没有安装对应的archive,这个命令可以在上面的步骤1看到说明，我们只需要按说明来生成一个archive就行

```elixir
> mix archive #查看当前install了什么archive
* hex-0.11.1.ez
Archives installed at: /Users/zhongwen/.mix/archives

> mix archive build  #它会生成一个App-Version.ez文件
Compiled lib/mix/lifelog.start.ex
Generated archive "lifelog-0.0.1.ez" with MIX_ENV=dev

> mix archive.install lifelog-0.0.1.ez                                 
Are you sure you want to install archive "lifelog-0.0.1.ez"? [Yn] y
* creating /Users/zhongwen/.mix/archives/lifelog-0.0.1.ez

> mix archive 
* hex-0.11.1.ez
* lifelog-0.0.1.ez
Archives installed at: /Users/zhongwen/.mix/archives
```
mix archive build如果不指定文件名并且在当前目录有mix.exs文件的话，会根据它里面的app 和 version生成 App-Version.ez,

这下我们就可以到处运行
```elixir
> mix lifelog.start
```
![Elixir003_001](http://7fveua.com1.z0.glb.clouddn.com/elixir003_01.png)

## 补充说明
以上已能完整的创建并安装一个archive啦，但是上面有一个比较有意思的函数，我们这个例子中并实际用到，但却是一个非常有用的函数！

```elixir
Interactive Elixir (1.2.2) - press Ctrl+C to exit (type h() ENTER for help)
iex(1)> OptionParser.parse(["--debug"])
{[debug: true], [], []}
iex(2)> OptionParser.parse(["--source", "lib"])
{[source: "lib"], [], []}
iex(3)> OptionParser.parse(["--source", "lib", "test/test.exs", "--verbose"])
{[source: "lib", verbose: true], ["test/test.exs"], []}
```
它可以把命令行的参数args转化为一个keywords list.

更多详细的例子可以通过（doc里面的例子更加全面）
```elixir
iex(4)> h OptionParser.parse
```
## Resources
1. [Erlang Archive Format](http://www.erlang.org/doc/man/code.html)

2. [Mix.Tasks.Archive.Build Documentation](http://elixir-lang.org/docs/master/mix/Mix.Tasks.Archive.Build.html)

3. [The Phoenix installer archive application](https://github.com/phoenixframework/phoenix/tree/master/installer) 这个看上去也很棒！

4. 以上提到的所有代码修改: [增加Task 并生成archive](https://github.com/zhongwencool/lifelog/commit/8ee9c4b38c683f863eb9777b2e7a3b63f46d06ad)

