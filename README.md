 # 在LNMP下部署最新SSPanel_Dev并与XrayR后端对接的教程（截至2023年7月）
---
### 前言：
### 过去的一年时间，基于SSPanel搭建的飞机场发生了翻天覆地的变化。首先SSPanel完全更换了框架，界面大改，然后V2Ray-Poseidon作者删库跑路，留下我独自在风中凌乱。再加上特定时期总会有那么一批SSL端口甚至服务器IP被毙掉、SSPanel的Wiki其实相当笼统，作者团队也相当高傲（个人觉得差不多相当于“有问题那都是你不会用”）、面板所需要的依赖，以及依赖的依赖总会有所变化、面板对应的数据库结构也总在变化……总之直到憋出这篇个人认为可以分享的经验为止，并不是什么愉快、舒适的过程。但无论如何，这篇Readme还是憋出来了。作为一名学英语的文科生，我会用我可怜的知识储备，尽量将这一路过来的酸甜苦辣给大家分享透彻。当然了，本文所提到的各种解决方案、代码等等，也并非100%按照官方说明和规范进行。有经验的同学可以在我的基础上自行研究更好的内容，而像我一样只会复制粘贴的同学，本文也应该能确保你的项目可以正常跑起来了。
这里再次提醒：本人搭建机场并非出于商业目的，而是旨在建立一个图形化、规范化、方便管理的平台，用来统一管理多个服务器节点，并且可以在确保不盈利的情况下，通过象征性收取一些费用，与自己身边的熟人朋友进行分享（本人面板下的所有用户交的一年费用，甚至不够抵服务器半年的租金）。因此，追求回本无可厚非，也不会带来什么法律责任，但请不要试图用租赁机场的方式赚钱，出租时也请确保对方人品可靠，用途可控。因为底线不在于你是否绕过了GFW，而在于你有没有因此赚钱，以及你有没有在绕过GFW后做一些违反法律的事情。

如果你对本文涉及到的前后端等程序感兴趣，请访问以下链接。在此也特别感谢各路大神提供的各种教程文章。

SSPanel-Uim：https://github.com/Anankke/SSPanel-UIM

XrayR：https://github.com/XrayR-project

LNMP：https://lnmp.org

---

### 目录：
[一、安装LNMP的要点][安装LNMP的要点]
   
   [1. Nginx可选模块的安装][Nginx可选模块的安装]
   
   2. PHP必要模块的安装
   3. PHP已禁用功能的恢复
   4. Redis的安装及配置

二、部署Dev分支SSPanel-Uim的要点
   
   1. 个人的一些有区别的代码
   2. 旧版数据库迁移的方式
   3. 关于IPv6环境存在的问题
   4. 移除跨目录访问限制

三、部署XrayR后端的要点

   1. XrayR配置文件
   2. Nginx配置文件
   3. 节点信息填写以及Custom-config
   4. 简单分析XrayR log

四、服务器拥塞算法的选择

五、关于CDN个人的一些反馈和看法

六、附录

---

首先，我个人推荐使用FinalShell作为SSH工具连接服务器操作。该软件提供了比较完善的图形化界面和直观的文件管理工具，可以让你在不输入命令的情况下轻松编辑文件。详情请访问：https://www.hostbuf.com/

如果你不喜欢FinalShell，也可以自行选择熟悉的SSH客户端进行操作。本教程将部分参照FinalShell的操作方式进行。

本教程将基于RockyOS 8（即CentOS 8的平替版本）进行编写。使用Debian或Ubuntu的同学请自行替换命令内容。

全新的VPS在操作前，建议先安装一些基本的工具和依赖：
```
sudo yum -y install git wget vim nano socat
```

在后续的内容中，默认你已经获得了系统的root权限，所有操作在root账户下进行。如果没有进入root账户，请自行在命令前添加sudo，或参照附录获得root权限。

#### 一、安装LNMP的要点
#### 1.Nginx可选模块的安装

总体来说，LNMP本身的安装没有什么难度，但为了后续操作，我们可能还需要修改一些代码。但这一步也不一定每位同学都需要做，你可以根据我下面的说明自行选择。
首先下载LNMP并解压到当前目录：
```
wget http://soft.vpser.net/lnmp/lnmp2.0.tar.gz -O lnmp2.0.tar.gz && tar zxf lnmp2.0.tar.gz && cd lnmp2.0
```
进入lnmp目录后，修改当前目录下的lnmp.conf：
```
vim lnmp.conf
```
为Nginx_Modules_Options添加“--with-stream_ssl_preread_module”命令。

SSL_Preread模块可用于希望V2Ray和Trojan并存、且不会与Nginx监听端口冲突的情况。但如果你不需要V2Ray和Trojan共存，则不需要进行任何修改：
```
Download_Mirror='https://soft.vpser.net'

Nginx_Modules_Options='--with-stream_ssl_preread_module'
PHP_Modules_Options=''
略
```
保存退出文本编辑器（vim使用英文冒号+wq回车，FinalShell可以直接Ctrl+S保存，以后不再赘述）。

然后编辑include目录下的main.sh，参照本Repo提供的文件，将1GB内存检测的代码注释掉，避免低配置VPS在编译安装MariaDB等时报错：
```
（约第115行）
#    if [ "${Bin}" != "y" ] && [[ "${DBSelect}" =~ ^5|[7-9]|10$ ]] && [ $(free -m | grep Mem | awk '{print  $2}') -le 1024 ]; then
#        echo "Memory less than 1GB, can't install MySQL 8.0 or MairaDB 10.3+!"
#        exit 1
#    fi
略
```
当然，你也可以直接下载本Repo中的文件对它们直接替换，但个人不推荐这样操作。因为LNMP安装包的版本一直在更新，代码也可能会有细微变化，直接替换可能会导致不可预知的后果。

#### 2.PHP必要模块的安装

根据最新的Dev分支SSPanel-Uim的要求，面板所在服务器的PHP必须要安装并启用以下模块，否则部署过程一定会报错：
```
curl fileinfo gd mbstring xml opcache zip json bz2 bcmath redis
```
以上模块除了Redis以外，PHP源码内都已经包含，可以很轻松地安装并启用。Redis将会单独在后文讲解。具体方法有几种：

**方案1：** 如果你还没有安装LNMP，需要全新安装，则参照上一点修改lnmp.conf，并在PHP_Modules_Options后加入以下内容：
```
PHP_Modules_Options='--enable-curl --enable-fileinfo --enable-gd --enable-mbstring --enable xml --enable-opcache --enable-zip --enable-zip --enable-json --enable-bz2 --enable-bcmath'
```
保存并退出lnmp.conf。

此外，低配置VPS请务必设置好SWAP，即虚拟内存，否则极其容易安装报错。你可以通过swapon -s命令查看当前系统是否存在虚拟内存，如果没有，请参照附录内容手动添加。
以上文件编辑好后，确保当前处在lnmp解压目录下，执行安装脚本：
```
./install.sh lnmp
```
安装时，PHP选择8.1或8.2，MariaDB选择10.11，数据库密码自行设定，其余保持默认，直接开始编译安装即可。作为参考，目前个人使用MariaDB 10.11.2和PHP 8.1.18的组合。

**方案2：** 如果你已经安装了LNMP，但版本号不符合要求（SSPanel-Uim要求PHP 8.x以上，以及MariaDB 10.3以上），则必须要进行更新，否则部署面板和数据库时一定会报错。此时我们可以顺手添加好PHP的模块。

方法参照方案1，修改好lnmp.conf后，在lnmp目录下执行以下命令升级PHP即可：
```
./upgrade.sh php
```
此时输入你想要升级到的PHP版本（见PHP官网下载页面），脚本会自动完成剩余的操作。MariaDB的升级方式基本相同，只需要把php改成mariadb，输入数据库密码和目标版本号即可。

**方案3：** 如果你已经安装了LNMP，且版本号也符合要求，不想重新编译安装一遍的，则需要麻烦一些，手动编译各个模块并添加进PHP中。具体方法如下：

首先进入LNMP一键安装包中PHP源码所在的目录，此处以LNMP 1.9为例：
```
cd lnmp1.9/src
ls
```
此时查看目录下是否有PHP对应的源代码压缩包，并检查版本号。如果没有压缩包，或版本号不满足要求（PHP 8.0以上），则需要手动下载源码压缩包并解压缩，此处以PHP 8.1.21为例：
```
wget https://www.php.net/distributions/php-8.1.21.tar.gz
tar -zxvf php-8.1.21.tar.gz
cd php-8.1.21/ext
```
分别进入前文对应的模块文件夹，并执行以下命令，此处以curl为例：
```
cd curl
/usr/local/php/bin/phpize
./configure --with-php-config=/usr/local/php/bin/php-config
make && make install
```
其余模块就是进入的文件夹名称不同，往后执行的命令全部都相同。

在除了Redis模块都都安装完毕后，我们需要再对php.ini进行配置，以便让PHP加载这些模块：
```
vim /usr/local/php/etc/php.ini
```
找到Dynamic Extensions一节，将指定模块前的“;”号去掉，例如：
```
（约第929行）
extension=curl
;extension=ffi
;extension=ftp
extension=fileinfo
extension=gd
extension=mbstring
extension=xml
略
```
如果如果发现extension后没有上述安装的模块，你可以手动插入一行并填写好模块名称即可。

#### 3.PHP已禁用功能的恢复
这个其实是安装SSPanel时老生常谈的步骤了。总体来说就是要启用一些PHP出于安全考虑默认禁用的功能。启用它们对安全性有多大影响我不得而知，但如果不启用的话，很有可能报502 Bad Gateway或其他错误。

同样的，还是编辑php.ini，找到disable_functions，将需要打开的功能删掉，这里直接给出改好以后的代码：
```
（约第323行）
disable_functions = passthru,system,chroot,chgrp,chown,ini_alter,ini_restore,dl,openlog,syslog,symlink,popepassthru,stream_socket_server
```
保存并退出php.ini，自行决定是否需要lnmp restart。

#### 4.Redis的安装及配置
新版SSPanel-Uim需要依赖Redis实现订阅下发、用户资料编辑（非管理员界面）等功能。如果不安装，在访问部分面板页面时会报HTTP 500错误。开启面板debug模式后会发现诸如“Could not connect to Redis at 127.0.0.1:6379: Connection refused”的提示。可惜的是，在SSPanel-Uim的Wiki中并没有提到这点，似乎他们是按照稳定版的内容去写的wiki，甚至连手动在CentOS中安装的章节都删掉了。

Redis默认不包含在PHP的源码中，需要手动下载并解压。这里选择下载到LNMP安装包的PHP源码目录内：
```
cd lnmp1.9/src/php-8.1.21/ext
wget https://pecl.php.net/get/redis-5.3.7.tgz && tar -zxvf redis-5.3.7.tgz && cd redis-5.3.7
```
参照第2点的方法，编译并安装redis模块。安装完成后，为php.ini添加extension：
```
extension=/usr/local/php/lib/php/extensions/no-debug-non-zts-20210902/redis.so
```
在实践过程中，我发现直接使用“extension=redis”并不能让PHP调用redis模块，因此在此采用了精确位置进行调用。注意：“no-debug-non-zts-20210902”这个目录名称可能会根据你的PHP版本发生改变，请在Redis模块编译并安装完成后留意结尾的信息，如果有不同的需要进行替换。

PHP模块安装完成后，我们还需要单独安装Redis-server程序。实践中发现，基于源码安装的Redis-server似乎找不到安装的位置，但命令行可以执行，因此解压到的目录请自行抉择，这里依旧以/root目录为例：
```
cd ~
wget http://download.redis.io/releases/redis-7.2-rc3.tar.gz && tar -zxvf redis-7.2-rc3.tar.gz
sysctl vm.overcommit_memory=1
```
此时先不急着make，因为Redis在后续的make test中，会提示需要Python3，所以首先需要安装（已安装的可以跳过）：
```
wget https://www.python.org/ftp/python/3.11.4/Python-3.11.4.tgz && tar -zxvf Python-3.11.4.tgz && cd Python-3.11.4.tgz
yum install -y zlib*
yum -y install zlib-devel bzip2-devel openssl-devel ncurses-devel sqlite-devel readline-devel tk-devel gdbm-devel db4-devel libpcap-devel xz-devel
```
这里插一嘴，上述yum install的依赖也是我复制来的，不知道具体作用，也不清楚有哪些还确实存在。实践过程发现部分包是没法安装的，但似乎没有影响Python3的安装使用，所以报错就让它报错吧。
```
./configure --prefix=/usr/local/python3 --with-ssl
make && make install
```
在安装结束后，创建软链接：
```
ln -s /usr/local/python3/bin/python3 /usr/bin/python3
ln -s /usr/local/python3/bin/pip3 /usr/bin/pip3
```
安装好Python3后，回到刚才的Redis源码目录，并准备好编译安装：
```
cd ~/redis-7.2-rc3
make
make test
make install
```
其中，三个make需要分别执行，至于make test做不做自行决定，这里Redis在make完成后会提示强烈推荐你make test，所以也就顺带进行了。make test过程比较久，中途还会报一些错误，这里暂时不用管，最后make install成功即可。

安装完成后，留在原地，修改redis.conf，修改以下几处地方：
```
（约第108行）
# By default protected mode is enabled. You should disable it only if
# you are sure you want clients from other hosts to connect to Redis
# even if no authentication is configured.
protected-mode no
略
（约第413行）
# Set the local environment which is used for string comparison operations, and 
# also affect the performance of Lua scripts. Empty String indicates the locale 
# is derived from the environment variables.
locale-collate "en_US.UTF-8"
略
（约第577行）
# Note: read only replicas are not designed to be exposed to untrusted clients
# on the internet. It's just a protection layer against misuse of the instance.
# Still a read only replica exports by default all the administrative commands
# such as CONFIG, DEBUG, and so forth. To a limited extent you can improve
# security of read only replicas using 'rename-command' to shadow all the
# administrative / dangerous commands.
replica-read-only no
```
其中，protected-mode和replica-read-only要设置成no，以避免后续出现“You can't write against a read only replica”的错误（非debug模式下也会HTTP 500）。locale-collate指定好语言和编码，避免启动redis-server时出现“redisFailed to configure LOCALE for invalid locale name”的报错（这也是之前make test时通常会出现的问题）。

保存并退出redis.conf，执行以下命令启动Redis：
```
redis-server /root/redis-7.2-rc3/redis.conf
```
如果正确出现Redis的logo和其他信息，则证明Redis服务器端正常启动。但此时Redis程序会保持前台运行，如果Ctrl+C的话会直接终止进程。所以我们还要回到redis.conf中配置后台运行：
```
（约第306行）
# By default Redis does not run as a daemon. Use 'yes' if you need it.
# Note that Redis will write a pid file in /var/run/redis.pid when daemonized.
# When Redis is supervised by upstart or systemd, this parameter has no impact.
daemonize yes
略
```
保存退出后再次执行上述命令运行Redis，此时不应有任何提示，但Redis应该正常运行在后台了。我们可以通过以下命令来简单验证：
```
redis-cli
```
如果正常出现“127.0.0.1:6379>”的命令行，则证明Redis启动成功，此时可以输入role来顺便查看保护模式是否已关闭：
```
127.0.0.1:6379> role
1) "master"
2) "192.168.40.37"
3) (integer) 8886
4) "connecting"
5) (integer) -1
```
理论上说，在关闭保护模式后role会从slave（从）变成master（主），但在实践中发现，它似乎还是会自动变回slave。但鉴于之前已经配置了相应的内容，只要后续SSPanel-Uim不出现HTTP 500报错（验证方式：登录后不进入管理后台，点击“我的-资料修改”，查看页面是否正常显示），订阅下发也正常，则可以正常使用。

如果一定需要改变，[请参照此文章](https://blog.csdn.net/huojiahui22/article/details/122448293)来解决slave的问题。注意：文章中命令行内的IP地址根据你自己输入role后得出的IP地址进行修改，比如我这里的192.168.40.37，端口号6379不变。



<!-- 文内引用链接 -->
[安装LNMP的要点]: ./README.md#一安装LNMP的要点
[Nginx可选模块的安装]: ./README.md#1nginx可选模块的安装
