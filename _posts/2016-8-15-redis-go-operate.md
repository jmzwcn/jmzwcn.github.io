---
tags: go
layout: post
title: Golang操作Redis数据库的一点实践
category: Distributed
---
Redis是一个开源的使用C语言编写、支持网络、可基于内存亦可持久化的日志型、Key-Value数据库，并提供多种语言的API。

Redis也是一种基于key/value的NoSql数据库，适用的场景包括：储存用户信息，比如会话、配置文件、参数、购物车等等。这些信息一般都和ID（键）挂钩，这种情景下键值数据库是个很好的选择。

<!--more-->
Redis 有三个主要使其有别于其它很多竞争对手的特点：

    Redis是完全在内存中保存数据的数据库，使用磁盘只是为了持久性目的； 
    Redis相比许多键值数据存储系统有相对丰富的数据类型； 
    Redis可以将数据复制到任意数量的从服务器中； 


下面列出如何在Golang语言中的操作代码



`代码`：

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

