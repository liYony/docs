# Linux系统介绍

## 1 文件系统简单介绍

```
 /
 ├── bin          所有用户都可以使用的、基本的命令
 ├── boot         启动文件，比如内核等
 ├── dev          设备文件，Linux特有的
 ├── etc          配置文件
 ├── home         家目录
 │   ├── book     用户book的家目录
 ├── lib          库
 ├── media        插上U盘等外设时会挂载到该目录下
 ├── mnt          用来挂载其他文件系统
 ├── opt          Optional，可选的程序
 ├── proc         用来挂载虚拟的proc文件系统，可以查看各进程(process)的信息
 ├── root         root用户的家目录
 ├── sbin         基本的系统命令，系统管理员才能使用
 ├── sys          用来挂载虚拟的sys文件系统，可以查看系统信息：比如设备信息
 ├── tmp          临时目录，存放临时文件
 ├── usr          Unix Software Resource, 存放可分享的与不可变动的的数据
 │   ├── bin      绝大部分的用户可使用指令都放在这里(与开机无关), /bin中的命令跟开机有关
 │   ├── games    游戏
 │   ├── include  头文件
 │   ├── lib      库
 │   ├── local    系统管理员在本机自行安装、下载的软件
 │   ├── sbin     非系统正常运作所需要的系统命令
 │   ├── share    放置共享文件的地方, 比如/usr/share/man里存放帮助文件
 │   └── src      源码
 └── var          主要针对常态性变动的文件，包括缓存(cache)、log文件等
```

## 2 ubuntu20 配置Linux开发环境

### 2.1 固定IP

[ubuntu20设置固定ip](http://t.csdnimg.cn/etRPS)

### 2.2 开启ssh

```shell
sudo apt-get install openssh-server
```