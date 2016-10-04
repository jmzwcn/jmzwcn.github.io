---
tags: go
layout: post
title: OpenWrt创建一个自由上网的环境
category: Language
---
学习GOLANG，不少依赖都被Block,这种现实需求较迫切。

首先需要在国外买个VPS，一个有独立IP的虚拟机，装OS(CentOS,Ubuntu等)，
<!--more-->
之后安装SS服务端，以Ubuntu16.04为例，
安装SS:apt-get install shadowsocks
开机启动:systemctl enable shadowsocks
配置
vi /etc/shadowsocks/config.json

{
    "server":"*.*.*.*",
    "server_port":8388,
    "local_address": "127.0.0.1",
    "local_port":1080,
    "password":"************",
    "timeout":300,
    "method":"aes-256-cfb",
    "fast_open": false,
    "workers": 1
}

启动:systemctl start shadowsocks

现在来配置客户端(OpenWrt端)，
电脑下载shadowsocks-libev和luci-app-shadowsocks,scp到OpenWrt的/tmp目录
opkg install shadowsocks-libev
opkg install luci-app-shadowsocks
安装libev前如果提示不满足依赖，执行opkg update再安装
配置(ss版本更新了，书写形式有变化)

config transparent_proxy                                                                                                                       
        option local_port '1234'                                                                                                               
        option main_server 'cfg084a8f'                                                                                                         
        option udp_relay_server 'same'                                                                                                         
                                                                                                                                               
config socks5_proxy                                                                                                                            
        option local_port '1080'                                                                                                               
        option server 'cfg084a8f'                                                                                                              
                                                                                                                                               
config port_forward                                                                                                                            
        option local_port '5300'                                                                                                               
        option destination '8.8.4.4:53'                                                                                                        
        option server 'cfg084a8f'                                                                                                              
                                                                                                                                               
config servers                                                                                                                                 
        option auth '0'                                                                                                                        
        option fast_open '0'                                                                                                                   
        option server_port '8388'                                                                                                              
        option server '*.*.*.*'                                                                                                          
        option timeout '600'                                                                                                                   
        option password '***'                                                                                                           
        option encrypt_method 'aes-256-cfb'                                                                                                    
                                                                                                                                               
config access_control                                                                                                                          
        option self_proxy '1'                                                                                                                  
        option wan_bp_list '/dev/null'                                                                                                         
        option lan_ifaces 'br-lan_guest'                                                                                                       
        option lan_target 'SS_SPEC_WAN_AC'           

说明，主要和Server端保持一直就行，在luci界面里操作非常简单
随路由器自启:/etc/init.d/shadowsocks enable
启动SS:/etc/init.d/shadowsocks start

为了避免DNS污染，不添加任何过滤条件，所有对lan_guest接口访问通通SS代理 

同时创建另一个wifi信号(OpenWrt_guest)，走lan_guest接口，关于如何在OpenWrt里创建两wifi信号，参考已有的配置，很容易实现(/etc/config/wireless,/etc/config/network和/etc/config/firewall)

这样，我们就有了一个方便的自由上网环境了，双信号(OpenWrt/OpenWrt-guest)，按需选择，自由切换。






参考网址

* [2016年国外支持Alipay(支付宝)付款VPS收集](http://www.zhujiceping.com/17104.html)

* [在 OpenWRT 上使用 ShadowSocks 建立透明代理](http://undownding.me/2015/02/10/use-shadowsocks-on-openwrt/)


