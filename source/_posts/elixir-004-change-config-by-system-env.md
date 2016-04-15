---
title: 通过系统环境变量灵活管理Config
date: 2016-03-14 21:18:04
categories: Elixir
tags: [config, mix]
---
在elixir的config中我们有时会使用的到一些不想暴露出来的配置项，常用的作法是如[Phoenix](https://github.com/phoenixframework/phoenix)
<!-- more -->

```elixir
#config/prod.exs
use Mix.Config
...
# Finally import the config/prod.secret.exs
# which should be versioned separately.
import_config "prod.secret.exs"
```

在上面，我们可以把不应该暴露的项（如数据库passwd，auth_token...这些很重要的项）都写在prod.secret.exs（链接可见例子）中

我们再把prod.secret.exs这个文件不要加到项目的版本管理中， 单独开一个git仓库管理。

但是有时我们并不想再这样，还有一种方法（而且感觉比上面这种方法更好用）

就是把所有的应该写在prod.sesret.exs的项都写到系统的环境变量中。
```elixir
#config/prod.exs
use Mix.Config
....
config :application_name, ApplicationName.Repo,
  adapter: Ecto.Adapters.MySQL,
  username: System.get_env("PROD_REPO_USERNAME"),
  password: System.get_env("PROD_REPO_PASSWORD"),
  database: System.get_env("PROD_REPO_DATABASE"),
  hostname: System.get_env("PROD_REPO_HOSTNAME")
...
```
这时只需要在服务器上
```elixir
#prod.env
export PROD_REPO_USERNAME='username'
export PROD_REPO_PASSWORD='loveyou'
export PROD_REPO_DATABASE='database'
export PROD_REPO_HOSTNAME='11.11.11.11'
```
只需要先
```elixir
>source prod.env
>iex -S mix
```
这样就可以通过环境变量来管理elixir的配置啦。

