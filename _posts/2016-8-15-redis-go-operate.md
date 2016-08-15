---
layout: post
title: Golang操作Redis数据库的一点实践
categories: Distributed
tags: go nosql memory
---

Redis是一个开源的使用C语言编写、支持网络、可基于内存亦可持久化的日志型、Key-Value数据库，并提供多种语言的API。本文列出如何在Golang语言中的操作代码


```go	
package main
import (
 "fmt"
 "log"
 "redis"
)
func main() {
 //DefaultSpec()创建一个连接规格
 spec := redis.DefaultSpec().Db(0).Password("");
 //创建一个新的syncClient，并连接到Redis的服务器上使用,指定ConnectionSpec接口。
 client, err := redis.NewSynchClientWithSpec (spec);
 if err != nil {
  log.Println ("连接服务器失败>", err);
  return
 }
 dbkey := "GAME:TEST:info";
 value, err := client.Get(dbkey);
 if err!= nil {
  log.Println ("error on Get", err);
  return
 }
 //
 if value == nil {
  value :=[]byte("Hello world!");
  client.Set(dbkey, value);
  fmt.Printf("插入数据>%s \n",value)
 } else {
  fmt.Printf("接收到数据>%s \n",value);
  //return;
 }
}
```

