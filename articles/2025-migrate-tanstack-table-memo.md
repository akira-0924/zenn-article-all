---
title: "react-table(v7)からtanstack-table(v8)に移行するときのメモcreateColumnHelper"
emoji: "😀"
type: "tech"
topics: [React, TypeScript]
published: false
---

react-table(v7)からtanstack-table(v8)に移行する作業をしていて、`createColumnHelper`周りで小技を使ってみたのでその備忘録です。

https://tanstack.com/table/v8/docs/guide/migrating#update-column-definitions


## やりたかったこと

**明示的に型を定義せずともtableで使用するcolumnの型をうまく活用したい**

[tanstack-table](https://tanstack.com/table/latest)を使用するにあたってcolumnの定義をするときに`createColumnHelper`を使用することが推奨されています。

![a](/images/migtate-tanstack01.png)

この部分ですね。