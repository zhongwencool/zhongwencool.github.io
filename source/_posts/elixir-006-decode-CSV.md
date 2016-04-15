---
title: Elixir decode&encode CSV 例子
date: 2016-04-04 01:31:45
categories: Elixir
tags: csv
---

[CSV](https://en.wikipedia.org/wiki/Comma-separated_values)有时也称为字符分隔值，因为分隔字符也可以不是逗号），其文件以纯文本形式存储表格数据（数字和文本）。纯文本意味着该文件是一个字符序列，不含必须像二进制数字那样被解读的数据。CSV文件由任意数目的记录组成，记录间以某种换行符分隔；每条记录由字段组成，字段间的分隔符是其它字符或字符串，最常见的是逗号或制表符。通常，所有记录都有完全相同的字段序列。

<!-- more -->
Elixir有好几个CSV处理的库([cesso](https://github.com/meh/cesso), [csv](https://github.com/beatrichartz/csv), [csvlixir](https://github.com/jimm/csvlixir), [ex_csv](https://github.com/CargoSense/ex_csv)), [beatrichartz/cvs](https://github.com/beatrichartz/csv)写得特别好,用了stream, test coverage(100%),而且有benchmark.

```elixir
mix new csv_play
cd csv_play
emacs mix.exs
```
```
defp deps do
    [{:csv, "~> 1.2.0"},
     {:faker, "~>0.5.1"}
    ]
end
```
## Faker测试数据
[faker](https://github.com/igas/faker)用来假造一些测试数据,接下来我们搞一个脚本生成csv数据
```elixir
mix deps.get
mkdir scripts
emacs scripts/generate_csv_data.exs
```

```elixir
Faker.start
count = 10_000
headers = ~w(name company city)

data =  1..count
|> Enum.map(fn(_) ->
  [Faker.Name.first_name <> Faker.Name.last_name, Faker.Company.name, Faker.Address.city]
  end)
  
file = File.open!("sample_data.csv", [:write])
  
[headers | data]
|> CSV.encode
|> Enum.each(&IO.write(file, &1))
  
:ok = File.close(file)
```

```elixir
mix run scripts/generate_csv_data.exs
```
## CSV decode示例
我们来测试一下csv的header读得是否正确

```elixir
mkdir test/data/
mv sample_data.csv test/data/
emacs test/csv_play_test.exs
```
```elixir
defmodule CsvPlayTest do
  use ExUnit.Case
  @data_path "test/data/sample_data.csv"
  test "reading CSV as a list" do
    list =
    @data_path
    |> File.stream!
    |> CSV.decode
    |> Enum.to_list
    assert hd(list) == ["name", "company", "city"]
  end
end
```
通常我们的cvs数据量都是非常大的，所以我们使用stream一块一块的来处理成map。
```elixir
test "reading CSV as a map" do
  list = @data_path
  |> File.stream!
  |> CSV.decode(headers: true)
  |> Enum.to_list
                
  sorted_keys = list
  |> hd
  |> Map.keys
  |> Enum.sort
                                
  assert sorted_keys == Enum.sort(["name", "company", "city"])
end
```
当然你CSV decode时还可以指定分割符是什么（默认就是上面这种形式的逗号）
```elixir
CSV.decode(separator: ?\t)
```
## Resources
 1. [beatrichartz/csv](https://github.com/beatrichartz/csv)
 
 2. [igas/faker](https://github.com/igas/faker)

