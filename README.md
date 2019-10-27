Open-channel
==========



# 概述
-----
Open-channel是一个批量管理服务器的工具，与ansible、saltstack功能类似，基于golang开发，其核心组件包括channel-master、channel-agent、cli，与这些运维工具相比，它具有如下特点：

1. 运维方便，其客户端管理工具cli提供全方位的系统运行时信息，运维可以快速定位
2. 稳定，系统本身支持高可用和负载均衡，支持过载保护，数据库弱依赖，服务端热重启
3. 配置动态加载
4. 安全，服务端需要认证token
5. 完善的自监控



# 安装
## 服务端
-----



```bash
mkdir -p /opt/channel-master
cd /opt/channel-master && wget https://github.com/zqwlai/open-channel/releases/download/1.0.0/channel-agent-1.0.0.tar.gz && tar zxvf channel-agent-1.0.0.tar.gz
#导入数据库表
mysql -h 127.0.0.1 -u root -p < scripts/mysql/channel-db-schema.sql 
#运行etcd
wget https://github.com/zqwlai/open-channel/releases/download/1.0.0/etcd && chmod +x etcd && nohup ./etcd --listen-client-urls http://0.0.0.0:2379 -advertise-client-urls http://0.0.0.0:2379 --data-dir=/var/lib/etcd &

#启动服务端
./control start

```

## 客户端

```

mkdir -p /opt/channel-agent
cd chanel-agent && wget https://github.com/zqwlai/open-channel/releases/download/1.0.0/channel-agent-1.0.0.tar.gz && tar -zxf channel-agent-1.0.0.tar.gz
#启动客户端
./control start
```
# 配置文件说明

## 服务端配置

----
配置文件需要满足以下的条件:

+ 配置文件必须和应用程序channel-master在同一目录
+ 配置文件命名，必须为 ```cfg.json```

配置文件的内容，如下

```go
{
  "debug": true,	//是否开启debug 
  "ip": "",         //不填写默认为对外通信IP，如果有VIP则填写VIP地址

  "http": {
    "listen": "0.0.0.0:8105"        //http监听端口
  },

  "etcd": {
    "addrs": ["127.0.0.1:2379"]         //etcd集群
  },

  "db": "root:123qwe@/channel_db?charset=utf8&parseTime=true",  //mysql连接信息
  "timeout": 600,                       //同步执行超时时间，单位秒
  "await_timeout": 1800,                //异步执行超时时间，单位秒
  "maxQps": 0                       //最大允许的Qps，为0表示不开启过载保护
  "script_dir": "/usr/local/scripts",       //脚本存放目录
  "logfile": "/var/log/channel-master.log"	//日志路径
}


```

## 客户端配置

----

配置文件的内容，如下

```go
{
  "debug": true,

  "http": {
    "listen": ":1990"       //自身监听的http端口

  },

  "etcd": {
    "addrs": ["127.0.0.1:2379"]     //etcd集群
  },

  "master": {
    "endpoints": ["192.168.1.5"],       //服务端地址，配置多个可以实现负载均衡
    "http_port": 8105,                  //服务端http端口
    "connTimeout": 2,					//单位秒
    "callTimeout": 5					//单位秒
  },

  "script_dir": "/tmp/scripts"      //脚本存放目录
  "logfile": "/var/log/channel-agent.log"	//日志路径
}

```


# cli（客户端管理工具）使用说明
cli作为一个可执行文件，可以管理用户、文件、执行命令/脚本、查看客户端/服务端运行状态、重载客户端/服务端配置等等

```
    ########  启动相关  ########
    cli stop|start|restart    #启动|停止|重启agent

    ########  状态相关  ########
    cli status                #客户端启动状态
    cli info                  #客户端状态统计
    cli agent_status          #本地客户端运行时信息
    cli master_status         #服务端运行状态
    cli online --limit m --page n -q xxx    #查看在线的机器
    cli reload_config --role=master|agent   #重载服务端|客户端配置（管理员）
    cli check -i ip（检测任意客户端ip） 

    ########  (命令/脚本)执行  ########
    cli shell -i ip -c cmd -t timeout -u user   #执行shell命令
    cli script -i ip -a args -f file -t timeout -u user --async(异步，可选) --callback =xxx(回调url，可选)  #执行脚本

    ########  文件相关    ######
    cli upload -f filename  #上传文件
    cli delfile -f filename  #删除文件
    cli download -f filename -d 远程目录(默认当前目录) #下载文件
    cli listfile    #列出本用户下的文件列表

    ########    用户相关    #######
    cli adduser -u username -p password     #创建新用户(管理员权限)
    cli chgpasswd -u username -p password  #更改用户密码(管理员权限)
    cli disableuser -u username #禁用用户(管理员权限)
    cli enableuser  -u username #启用用户(管理员权限)
    cli deluser  -u username #删除用户(管理员权限)
    cli listuser --limit m --page n -q xxx #查看用户(管理员权限)

```

# http api

## 执行命令

```python
#!/usr/bin/env python

import requests
import json

data = json.dumps({
    'module':'shell',
    'cmd': 'uptime',
    'hosts': '1.1.1.1,2.2.2.2',
    #'async': True,
    #'callback': 'your url'
})

headers = {'Content-Type': 'application/json', 'token':'abc'}
r = requests.post('http://master_ip:master_http_port/api/v1/execute', data=data, headers=headers)
print r.text

```


## 执行脚本

```python
#!/usr/bin/env python

import requests
import json

data = json.dumps({
    'module':'script',
    'cmd': 'uptime',
    'hosts': '1.1.1.1,2.2.2.2',
    'filename': '1.py',
    'args': ''
    #'async': True,
    #'callback': 'your url'

})

headers = {'Content-Type': 'application/json', 'token':'abc'}
r = requests.post('http://master_ip:master_http_port/api/v1/execute', data=data, headers=headers)
print r.text

```






