 # 在LNMP下部署最新SSPanel_Dev并与XrayR后端对接的教程（截至2023年11月）
---
### 前言：
### 过去的一年时间，基于SSPanel搭建的飞机场发生了翻天覆地的变化。首先SSPanel完全更换了框架，界面大改，然后V2Ray-Poseidon作者删库跑路，留下我独自在风中凌乱。再加上特定时期总会有那么一批SSL端口甚至服务器IP被毙掉、SSPanel的Wiki其实相当笼统，作者团队也相当高傲（个人觉得差不多相当于“有问题那都是你不会用”）、面板所需要的依赖，以及依赖的依赖总会有所变化、面板对应的数据库结构也总在变化……总之直到憋出这篇个人认为可以分享的经验为止，并不是什么愉快、舒适的过程。但无论如何，这篇Readme还是憋出来了。作为一名学英语的文科生，我会用我可怜的知识储备，尽量将这一路过来的酸甜苦辣给大家分享透彻。当然了，本文所提到的各种解决方案、代码等等，也并非100%按照官方说明和规范进行。有经验的同学可以在我的基础上自行研究更好的内容，而像我一样只会复制粘贴的同学，本文也应该能确保你的项目可以正常跑起来了。
这里再次提醒：本人搭建机场并非出于商业目的，而是旨在建立一个图形化、规范化、方便管理的平台，用来统一管理多个服务器节点，并且可以在确保不盈利的情况下，通过象征性收取一些费用，与自己身边的熟人朋友进行分享（本人面板下的所有用户交的一年费用，甚至不够抵服务器半年的租金）。因此，追求回本无可厚非，也不会带来什么法律责任，但请不要试图用租赁机场的方式赚钱，出租时也请确保对方人品可靠，用途可控。因为底线不在于你是否绕过了GFW，而在于你有没有因此赚钱，以及你有没有在绕过GFW后做一些违反法律的事情（比如诈骗、黑客攻击、钓鱼网站等）。

如果你对本文涉及到的前后端等程序感兴趣，请访问以下链接。在此也特别感谢各路大神提供的各种教程文章。

SSPanel-Uim：https://github.com/Anankke/SSPanel-UIM

XrayR：https://github.com/XrayR-project

LNMP：https://lnmp.org

---

### 目录：

准备工作

一、安装LNMP的要点
   
   1. Nginx可选模块的安装
   2. PHP必要模块的安装
   3. PHP已禁用功能的恢复
   4. Redis的安装及配置

二、部署Dev分支SSPanel-Uim的要点
   
   1. 个人的一些有区别的代码
   2. 旧版数据库迁移的方式
   3. 关于IPv6环境存在的问题
   4. Windows及Android客户端的文件包下载
   5. Clash及Clash兼容客户端的订阅下发问题


三、部署XrayR后端的要点

   1. 下载并安装XrayR
   2. 节点信息填写以及Custom-config
   3. XrayR配置文件
   4. Nginx配置文件
   5. 启动节点并查看log

四、服务器拥塞算法的选择

五、关于CDN个人的一些反馈和看法

六、附录

---

### 准备工作

首先，我个人推荐使用FinalShell作为SSH工具连接服务器操作。该软件提供了比较完善的图形化界面和直观的文件管理工具，可以让你在不输入命令的情况下轻松编辑文件。详情请访问：https://www.hostbuf.com/

如果你不喜欢FinalShell，也可以自行选择熟悉的SSH客户端进行操作。本教程将部分参照FinalShell的操作方式进行。

~~本教程将基于RockyOS 8（即CentOS 8的平替版本）进行编写。使用Debian或Ubuntu的同学请自行替换命令内容。~~ 本教程将基于Ubuntu 20.04/22.04系统进行编写，使用CentOS的同学请自行替换命令内容。由于个人比较喜欢用的Xanmod内核目前仅支持Ubuntu/Debian系统，所以转为使用该系统。实际上Ubuntu和CentOS最主要的区别也就是安装软件包的命令不太一样而已（一个是apt-get，一个是yum），代换不会有太多问题。

全新的VPS在操作前，建议先安装一些基本的工具和依赖：
```
sudo apt-get install git wget curl vim nano socat
```

在后续的内容中，默认你已经获得了系统的root权限，所有操作在root账户下进行。如果没有进入root账户，请自行在命令前添加sudo，或参照附录获得root权限。

#### 更换Xanmod内核 & 更改拥塞算法
Xanmod内核相比起官方使用的generic内核，执行效率更高，支持更多功能，内核版本也通常保持较新的水平，对于低配VPS和对速度、延迟要求较高的应用场景有不错的改善作用。可惜的是，Xanmod目前只支持Ubuntu/Debian系统，CentOS如果想用，只能手动安装5.x版本，或改用其他内核了。

另外，Xanmod在安装上也没有想象的简单，所以在这里将正确的操作步骤总结。

准备工作：导入GPG Pubkey

在终端中输入以下命令：
```
apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv-keys 86F7D09EE734E623
```
否则过后安装内核时会提示“NO_PUBKEY 86F7D09EE734E623”报错，导致内核安装失败，甚至在安装某些软件包时也会有这样的报错。

方案1：使用一键加速脚本（推荐）

使用加速脚本安装内核是比较方便的，它可以帮你安装好所有的依赖，减少出现问题的可能性。另外，后续我们需要修改拥塞（加速）算法时，使用一键脚本也更加简洁直观。

输入以下命令下载加速脚本：
```
wget -N --no-check-certificate "https://github.000060000.xyz/tcpx.sh" && chmod +x tcpx.sh && ./tcpx.sh
```
之后输入对应数字，安装Xanmod (main)并重启即可。如果自己喜欢别的选择，也可以自行操作。

安装后要注意，在脚本列出的内核列表中是否存在Xanmod，若没有，则安装失败，请参照方案2。

方案2：手动安装

以Xanmod-6.5.8-x64v3版本为例，分别下载内核镜像和headers：
```
wget https://zenlayer.dl.sourceforge.net/project/xanmod/releases/main/6.5.8-xanmod1/6.5.8-x64v3-xanmod1/linux-image-6.5.8-x64v3-xanmod1_6.5.8-x64v3-xanmod1-0~20231020.ga323bd9_amd64.deb

wget https://zenlayer.dl.sourceforge.net/project/xanmod/releases/main/6.5.8-xanmod1/6.5.8-x64v3-xanmod1/linux-headers-6.5.8-x64v3-xanmod1_6.5.8-x64v3-xanmod1-0~20231020.ga323bd9_amd64.deb
```
然后执行以下命令安装内核：
```
dpkg -i linux-image-6.5.8-x64v3-xanmod1_6.5.8-x64v3-xanmod1-0~20231020.ga323bd9_amd64.deb linux-headers-6.5.8-x64v3-xanmod1_6.5.8-x64v3-xanmod1-0~20231020.ga323bd9_amd64.deb
```
即“dpkg -i 内核镜像.deb headers.deb”。等待安装完成，重启即可。

如果你需要修改拥塞算法，请下载上述加速脚本，然后选择你想要的组合即可。个人比较喜欢的是BBR+FQ_PIE。不同的加速方案在不同的服务器上的表现也不尽相同（比如BBR2和BBRPlus在我的服务器上表现就很糟糕），因此请自己反复尝试，直到找到最适合自己的方案。

#### 添加虚拟内存
现在有很多VPS默认不会打开虚拟内存（swap），这样很容易导致在LNMP编译的过程中因内存不足而Error 1报错退出。我们可以输入以下命令检查是否存在swap分区：
```
swapon -s
```
如果输入命令后出现内容，例如：
```
Filename                                Type            Size    Used    Priority
/var/swapfile                           file            3071996 34208   -2
```
才代表已经启用了swap。如果什么都没有，证明没有打开。可以输入以下命令，给系统添加一个大约3GB的虚拟内存：
```
dd if=/dev/zero of=/var/swapfile bs=1024 count=3072000 && mkswap /var/swapfile && chmod -R 0600 /var/swapfile && swapon /var/swapfile && echo "/var/swapfile swap swap defaults 0 0" >> /etc/fstab
```
这个swap文件在创建后会保留在硬盘上，开机自动激活。如果你不需要了，可以参照[这篇文章](https://blog.csdn.net/ausboyue/article/details/73433990)来删除swap。

至此，前期准备过程已完成。


#### 一、安装LNMP的要点

#### 1. Nginx可选模块的安装
<details><summary>点击展开</summary>

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

SSL_Preread模块可用于希望V2Ray和Trojan并存、且不会与Nginx监听端口冲突的情况（有可能提升SSL端口的存活率，但未经验证）。但如果你不需要V2Ray和Trojan共存，可以不进行修改。本文也将只会使用V2Ray，具体原因后文解释：
```
Download_Mirror='https://soft.vpser.net'

Nginx_Modules_Options='--with-stream_ssl_preread_module'
PHP_Modules_Options=''
略
```
保存退出文本编辑器（vim使用英文冒号+wq回车，FinalShell可以直接Ctrl+S保存，以后不再赘述）。

然后编辑include目录下的main.sh，参照本Repo提供的文件，将1GB内存检测的代码注释掉，或者直接删除，避免低配置VPS在编译安装MariaDB等时报错：
```
（约第291行）
#    if [ "${Bin}" != "y" ] && [[ "${DBSelect}" =~ ^5|[7-9]|10$ ]] && [ $(free -m | grep Mem | awk '{print  $2}') -le 1024 ]; then
#        echo "Memory less than 1GB, can't install MySQL 8.0 or MairaDB 10.3+!"
#        exit 1
#    fi
```
当然，你也可以直接下载本Repo中的文件对它们直接替换，但个人不推荐这样操作。因为LNMP安装包的版本一直在更新，代码也可能会有细微变化，直接替换可能会导致不可预知的后果。

</details>

#### 2. PHP必要模块的安装
<details><summary>点击展开</summary>

根据最新的Dev分支SSPanel-Uim的要求，面板所在服务器的PHP必须要安装并启用以下模块，否则部署过程一定会报错：
```
curl fileinfo gd mbstring xml zend-opcache zip json bz2 bcmath sodium yaml redis
```
以上模块除了Redis以外，PHP源码内都已经包含，可以很轻松地安装并启用。此外，新版的PHP已经启用了上面大多数模块，如curl、xml、json、zip等等，具体可以在安装好LNMP后，访问服务器IP并点击phpinfo，或使用php -m来查看哪些模块已经启用。已经启用好的模块就不再需要安装了。

Redis将会单独在后文讲解。具体方法有几种：

**方案1：** 如果你还没有安装LNMP，需要全新安装，则参照上一点修改lnmp.conf，并在PHP_Modules_Options后加入以下内容：
### UPDATE：目前发现方案1、2似乎并不能在安装过程中编译安装上述模块（是我没研究明白），暂时建议参照方案3进行操作。
```
PHP_Modules_Options='--enable-curl --enable-fileinfo --enable-gd --enable-mbstring --enable-xml --enable-opcache --enable-zip --enable-bz2 --enable-bcmath'
```
保存并退出lnmp.conf。

此外，低配置VPS请务必设置好SWAP，即虚拟内存，否则极其容易安装报错。你可以通过swapon -s命令查看当前系统是否存在虚拟内存，如果没有，请参照附录内容手动添加。
以上文件编辑好后，确保当前处在lnmp解压目录下，执行安装脚本：
```
./install.sh lnmp
```
安装时，PHP选择8.1或8.2，MariaDB选择10.11，数据库密码自行设定，其余保持默认，直接开始编译安装即可。作为参考，目前个人使用MariaDB 10.11.2和PHP 8.2.6的组合。

**方案2：** 如果你已经安装了LNMP，但版本号不符合要求（SSPanel-Uim要求PHP 8.x以上，以及MariaDB 10.3以上），则必须要进行更新，否则部署面板和数据库时一定会报错。此时我们可以顺手添加好PHP的模块。

方法参照方案1，修改好lnmp.conf后，在lnmp目录下执行以下命令升级PHP即可：
```
./upgrade.sh php
```
此时输入你想要升级到的PHP版本（见PHP官网下载页面），脚本会自动完成剩余的操作。MariaDB的升级方式基本相同，只需要把php改成mariadb，输入数据库密码和目标版本号即可。

**方案3：** 如果你已经安装了LNMP，且版本号也符合要求，不想重新编译安装一遍的，则需要麻烦一些，手动编译各个模块并添加进PHP中。具体方法如下：

首先进入LNMP一键安装包中PHP源码所在的目录，此处以LNMP 2.0为例：
```
cd lnmp2.0/src
ls
```
此时查看目录下是否有PHP对应的源代码压缩包，并检查版本号。如果没有压缩包，或版本号不满足要求（PHP 8.0以上），则需要手动下载源码压缩包并解压缩，此处以PHP 8.2.6为例：
```
wget https://www.php.net/distributions/php-8.2.6.tar.gz
tar -zxvf php-8.2.6.tar.gz
cd php-8.2.6/ext
```
分别进入前文对应的模块文件夹，并执行以下命令，此处以curl为例：
```
cd curl
/usr/local/php/bin/phpize
./configure --with-php-config=/usr/local/php/bin/php-config
make && make install
```
其余模块就是cd进入的文件夹不同，往后执行的命令全部都相同。 **请再次注意PHP默认已经安装和启用的模块，避免重复安装。**

**特别提醒：yaml模块默认不包含在PHP安装包里，需要手动下载源码后编译安装，并且编译扩展之前，需要首先在系统中安装yaml程序包：**
```
wget https://pyyaml.org/download/libyaml/yaml-0.2.5.tar.gz
tar -zxvf yaml-0.2.5.tar.gz
cd yaml-0.2.5
./configure
make && make install
```

然后再下载和安装PHP的yaml模块：
```
wget https://pecl.php.net/get/yaml-2.2.3.tgz
tar -zxvf yaml-2.2.3.tgz
cd yaml-2.2.3
/usr/local/php/bin/phpize
./configure --with-php-config=/usr/local/php/bin/php-config
make && make install
```
**特别提醒2：安装sodium模块之前，必须先安装libsodium程序包：**
```
wget https://download.libsodium.org/libsodium/releases/libsodium-1.0.19.tar.gz
tar -xvzf libsodium-1.0.19.tar.gz
cd libsodium-stable
./configure
make && make install
```
之后再进入ext/sodium目录安装sodium模块。

在除了Redis模块都都安装完毕后，我们需要再对php.ini进行配置，以便让PHP加载这些模块：
```
vim /usr/local/php/etc/php.ini
```
找到Dynamic Extensions一节，将**刚才手动安装的模块**前的“;”号去掉，已默认安装开启的模块不用动，例如：
```
（约第929行）
;extension=curl
;extension=ffi
;extension=ftp
extension=fileinfo
;extension=gd
extension=mbstring
;extension=xml
;extension=json
略
```
sodium模块在稍微下面一点有写，直接去掉“;”即可。

如果如果发现extension后没有上述安装的模块，你可以手动插入一行并按照格式填写好模块名称即可。

对于opcache模块，由于它属于zend的模块，所以要在稍微下面一点的地方找到这行代码并去掉“;”号：
```
zend_extension=opcache
```
重启LNMP，若没有报错（“PHP message: PHP Warning:  Module "xxx" is already loaded in Unknown on line 0”的报错可以不用管，不影响正常启动，它的意思是模块已经默认加载，不用再手动改php.ini了。介意的话可以按照提示将“;”号加回去。），则证明模块已经安装成功。


</details>

#### 3. PHP已禁用功能的恢复
<details><summary>点击展开</summary>

这个其实是安装SSPanel时老生常谈的步骤了。总体来说就是要启用一些PHP或是LNMP出于安全考虑默认禁用的功能。启用它们对安全性有多大影响我不得而知，但如果不启用的话，很有可能报502 Bad Gateway或其他错误。

同样的，还是编辑php.ini，找到disable_functions，将需要打开的功能删掉，这里直接给出改好以后的代码：
```
（约第323行）
disable_functions = passthru,system,chroot,chgrp,chown,ini_alter,ini_restore,dl,openlog,syslog,symlink,popepassthru,stream_socket_server
```
保存并退出php.ini，自行决定是否需要lnmp restart。

</details>

#### 4. Redis的安装及配置
<details><summary>点击展开</summary>

新版SSPanel-Uim需要依赖Redis实现订阅下发、用户资料编辑（非管理员界面）等大多数功能。如果不安装，在访问部分面板页面时会报HTTP 500错误。开启面板debug模式后会发现诸如“Could not connect to Redis at 127.0.0.1:6379: Connection refused”的提示。

而官方Wiki中，只提及了安装Redis-server程序包，却并未提到要安装PHP模块。如果不进行安装，在部署面板时执行php composer.phar install时会报错，提示redis模块不存在。所以我们需要先手动安装PHP模块，再安装Redis-server。

Redis默认不包含在PHP的源码中，需要手动下载并解压。这里选择下载到LNMP安装包的PHP源码（以8.2.6版本为例）目录内：
```
cd lnmp2.0/src/php-8.2.6/ext
wget https://pecl.php.net/get/redis-5.3.7.tgz && tar -zxvf redis-5.3.7.tgz && cd redis-5.3.7
```
如果有其他想下载的版本，请自行修改命令中的版本号，以下不再赘述。

参照第2点的方法，编译并安装redis模块。安装完成后，在php.ini的Dynamic Extensions一节添加：
```
（约第929行）
extension=/usr/local/php/lib/php/extensions/no-debug-non-zts-20220829/redis.so
```
在实践过程中，我发现直接使用“extension=redis”并不能让PHP调用redis模块，因此在此采用了精确位置进行调用。注意：“no-debug-non-zts-20220829”这个目录名称可能会根据你的PHP版本发生改变，请在Redis模块编译并安装完成后留意结尾的信息，如果有不同的需要进行替换。

PHP模块安装完成后，我们还需要单独安装Redis-server程序。

首先向系统中导入GPG Pubkey：
```
curl -fsSL https://packages.redis.io/gpg | sudo gpg --dearmor -o /usr/share/keyrings/redis-archive-keyring.gpg
```
写入Redis官方源：
```
echo "deb [signed-by=/usr/share/keyrings/redis-archive-keyring.gpg] https://packages.redis.io/deb $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/redis.list
```
更新APT缓存，安装Redis-server并设置开机启动：
```
apt-get update
apt-get install redis
systemctl start redis-server
systemctl enable redis-server
```
安装完成后，输入以下命令打开Redis-server程序：
```
redis-cli
```
如果正常出现“127.0.0.1:6379>”的命令行，则证明Redis启动成功，此时可以输入role来顺便查看当前角色：
```
127.0.0.1:6379> role
1) "master"
2) (integer) 0
3) (empty array)
127.0.0.1:6379> exit
```
理论上说，全新安装的Redis-server角色默认就是“master”。如果出现“slave”，则有可能导致网站运行不正常。但只要不出现HTTP 500报错（验证方式：登录后不进入管理后台，点击“我的-资料修改”，查看页面是否正常显示），订阅下发也正常，则可以正常使用。

如果一定需要改变，或是确实存在问题，[请参照此文章](https://blog.csdn.net/huojiahui22/article/details/122448293)来解决。

</details>

#### 二、部署Dev分支SSPanel-Uim的要点

#### 1. 个人的一些有区别的代码
<details><summary>点击展开</summary>

首先，假设你已经拥有域名，并设置好解析，那么就可以使用lnmp vhost add命令添加一个Nginx主机了。具体内容不再赘述，确保创建一个数据库（本文名称将以sspanel为例），是否申请证书自行决定。添加好vhost后，我们可以先行修改网站对应的nginx配置文件，添加好伪静态。

这里你可以直接使用我提供的panel.example.com.conf中的代码完全代替原先的配置文件，使用时注意替换好代码中涉及的域名。按照上一版Wiki的指导，他们将配置文件修改得很简洁，但并不影响网站的正常工作。

你也可以直接在原有文件内容后添加以下代码：
```
        location / {
            try_files $uri /index.php$is_args$args;
        }


        location ~ \.php$ {
            include fastcgi_params;
            fastcgi_pass 127.0.0.1:9000;
            fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
            fastcgi_param PATH_INFO $fastcgi_path_info;
        }
```
其中，try_files是一直以来必须要有的伪静态，而fastcgi这一段我并不清楚具体用途，wiki也没有进行解释（而且按照他们的代码范例，直接跑起来是会报错的，必须要把pass改成如上的IP地址+端口号才行）。如果你怀疑fastcgi这部分的作用，可以不去添加它。

此外要记住，上述代码需要添加两次，一次放在监听80端口的节下面，一次放在443端口的节下面，此处不再赘述。

之后，修改好网站所在目录，在后面添加“/public”。假设你创建的网站目录为/home/wwwroot/sspanel：
```
root /home/wwwroot/sspanel/public; 
```
同样注意80和443两节的都要修改。

**警告：手动修改网站根目录将会导致LNMP提供的acme.sh证书自动续期失效，并且后续手动续期证书也会报404错误。在续期之前，请先将root中的“/public”去掉。个人不建议在lnmp vhost add时直接设置到public目录下，似乎会造成一些奇怪的故障。**

接下来，在部署网站之前，我们一定要移除LNMP默认的跨目录访问限制，否则浏览器一定会提示502 Bad Gateway。这里你可以使用LNMP安装包tools目录下的remove_open_basedir_restriction.sh（强烈推荐），也可以直接手动操作：
```
cd /home/wwwroot/sspanel
chattr -i .user.ini
rm .user.ini
sed -i 's/^fastcgi_param PHP_ADMIN_VALUE/#fastcgi_param PHP_ADMIN_VALUE/g' /usr/local/nginx/conf/fastcgi.conf
```
注：上述sed -i命令只需要最开始执行一次，以后创建其他网站只需要删除.user.ini即可，不需要再执行sed -i命令。

执行完毕后重启一次LNMP或PHP。

之后我们就可以参考Wiki内容直接部署网站了：
```
git clone -b dev https://github.com/Anankke/SSPanel-Uim.git .
wget https://getcomposer.org/installer -O composer.phar
php composer.phar
php composer.phar install --no-dev
```
若执行第一行命令时提示“fatal: destination path '.' already exists and is not an empty directory.”，你可以回到wwwroot目录下，使用rm -rf删除sspanel目录（记得先删除.user.ini，否则会提示访问拒绝），然后再mkdir sspanel，进入目录重新执行命令。

接下来，准备好面板程序的配置文件：
```
cp config/.config.example.php config/.config.php
cp config/appprofile.example.php config/appprofile.php
```
然后使用你喜欢的编辑器编辑config/.config.php。里面的内容主要根据自身情况设定，此处不再赘述。

编辑好后，执行php xcat向数据库导入表：
```
php xcat Migration new
php xcat Tool importAllSettings
php xcat Tool createAdmin
```
原文还有php xcat ClientDownload，作用是下载clash和singbox客户端压缩包到网站下，以便用户可以点击下载。但我是使用V2RayN和V2RayNG，所以过后再手动下载。另外，上述代码全部用于全新部署的情况。如果你是旧版升级或迁移，请先不要执行上述php xcat，具体参见下一节内容。

最后，使用crontab -e添加一个计划任务，以实现等级过期等检测（Ubuntu系统会提示你选择一个编辑方式，使用第一个nano即可）：
```
*/5 * * * * /usr/local/php/bin/php /home/wwwroot/sspanel/xcat  Cron
```
注意检查后半部分网站目录是否正确，添加好后保存退出（nano为Ctrl+O，回车保存，然后Ctrl+X退出）。

**最后一步非常重要！** 确保自己当前还在sspanel目录下，执行以下命令设置网站文件的权限：
```
chmod -R 777 *
chown -R root *
```
官方Wiki中，使用的是755权限和www-data用户。但实际上，我们在安装LNMP客户端时并没有生成www-data用户，而且在我的环境下，755权限还是有可能会报502 Bad Gateway，所以干脆偷懒直接使用root账户和777权限。理论上来说，这样设置是十分不安全的，但对于自己身边小规模的使用而言，也无所谓安不安全了。

如果有安全性方面的需求，请自行研究为Nginx、数据库等创建www账户的相关文章，本文不再赘述。

此时执行lnmp restart一次，重启LNMP。如果不报错，你应该就可以正常访问面板主页了。

</details>

#### 2. 旧版数据库迁移的方式
<details><summary>点击展开</summary>

对于已经安装了旧版SSPanel-Uim，由于各种原因需要更新的，你的痛苦旅程才刚刚开始。

Dev版的SSPanel一直在更新，而数据库里的表结构也一直在变化。比如user表一直在变化，原来还有的列到了新版就没有了；原来某个值可以为空，更新后变成不能为空了；原来用户密码还能md5加密，现在不行了……虽然官方提供了升级的方式（事实上目前版本的update.sh还是挺好用的，最要命的是以前的手动升级，我几乎就没成功过一次），但有时候版本间隔大了，还是很容易出现升级失败。而这时候，手动导入或直接使用旧版的数据库也必定导致网站崩溃的问题，因此，我们只能手动匹配新版数据库结构。

提醒：本节提到的数据库操作方式属于个人方法，不一定是最优解。有更好办法的同学可以提交issue。

**2.1 备份旧数据库**

这个没什么好说的，最简单的就是进phpmyadmin然后导出即可。本文不再赘述。

**2.2 将旧版数据库更名**

点击数据库，选择右边工具栏的“操作”，“重命名数据库为”一栏修改名称（例如sspanel_old），点击右侧执行即可。

**2.3 创建新数据库**

根据你在.config.php中数据库部分的配置，新建一个名称对应的数据库。

根据前文提到的部署方式，回到SSH中，进入网站所在目录，执行以下命令导入表：
```
php xcat Migration new
php xcat Tool importAllSettings
```
此时再回到phpmyadmin，你会发现新建的数据库中已经导入内容。

要无损迁移老数据库，我们需要使用到“操作-将数据表复制到-仅数据”功能。千万不可以使用“仅结构”或“结构和数据”，否则功亏一篑。手动迁移的原因就是新表的结构变了，所以不能带着结构迁移。

先从简单的开始。我个人只会迁移“announcement（公告）”、“node（节点）”、“product（商品）”和“user（用户）”表，其他均保留默认状态。因为我的面板不设置任何支付网关、审计规则等等，至于优惠码、礼品卡等也可以很方便地重新生成（使用记录对我来说没什么用处）。大家根据自己的情况检查各表内容，按照上述方法操作即可，原理都差不多。

以user表为例，进入旧版数据库的user表，点击“操作”，在“将数据表复制到”一栏中，选择好复制到的新数据库名称，下方点击“仅数据”，然后点击右边的“执行”。如果不出意外，一定会报错，提示“#1054 - Unknown column 'XXX' in 'field list'”。这就证明，旧表的结构中存在的列，新表中已经不存在，因此无法复制。

首先我们观察报错信息中提到的列的名称，例如“node_connector”，我们需要在旧的user表上方点击“结构”，找到“node_connector”，然后点击右边的“删除”。之后再重复一遍复制操作，根据新的报错信息再去“结构”内删除，直到复制成功为止。

其他数据表的操作几乎相同，这里不再重复。但值得注意的是，某些表里的内容可能会发生很大变化，有一定概率导致旧数据无法通用（即便复制成功。例如编写数据的语法发生改变时），当发现面板无法调用旧数据或数据异常时，通常也只能删掉旧数据，手动创建新数据。

**2.4 为特定列批量添加特定值**

当前版本相比起旧版（尤其是material主题的旧版），在user表中多出了几列内容，例如“use_new_shop”、“is_dark_mode”、“is_inactive”，以及“api_token”等。将旧表复制过来时，这些新增的列就会没有数据。通常，我们希望给这些列批量添加0或者1，或者是将别的列的内容直接粘贴过来。用户比较多的时候，使用SQL语句就会比较方便。

此处以“use_new_shop”列为例，我们希望所有用户的值都设置成1，那么可以点击上方的“SQL”，覆盖粘贴以下命令：
```
UPDATE user SET use_new_shop = 1
```
其公式为：UPDATE 表名 SET 列名 = 值。

而对于“api_token”一列，我们没办法很简单的安放数据，因为这里的内容跟“uuid”类似，是一串随机生成的代码。但既然两者比较相似，那么我们可以直接将“uuid”的内容套用到“api_token”上来，理论上可以使用。目前我个人还不是很了解这个API Token的作用，如果有特殊需求的同学，建议还是另想办法。

执行以下SQL语句：
```
UPDATE user SET api_token = uuid
```
其公式为：UPDATE 表名 SET 目标列名 = 源列名。

执行成功后，回到“浏览”页面，应该就能看到刚才操作的列已经有了我们想要的数据。

**2.5 新面板用户密码加密方式的应对**

目前最新版的SSPanel-Uim已经删除了用户密码的md5加密算法，转为默认使用bcrypt。而bcrypt加密后存储在user表中的密文显然不可能与md5相同。但也正因如此，直接迁移过来的user表会导致登录时提示密码错误。当务之急是解决admin账户无法登录的问题。

此时有两个解决方案，第一种很简单，在数据库中删掉admin账户，然后回到SSH，使用php xcat重建管理员账户：
```
php xcat Tool createAdmin
```
重建后的管理员账户只需要自己再设置想要的东西即可（比如等级、余额等）。

如果一定要保留原先管理员账户，可以到数据库user表下，将admin账户的ID改为其他数字（也可以连user_name和email一起改了，之后再改回来），然后使用php xcat Tool createAdmin再次创建一个管理员账户。此时数据库中将再次出现一个ID为1的管理员账户，将其pass（登录密码）的值粘贴到原先的Admin账户处，删掉新账户，将旧帐户ID再次改回1。此时就可以正常登录面板，且原先的所有账户数据都能得到保留。

对于其他用户的密码，我们无能为力，只能向用户索要密码，前往面板-站点管理-管理-用户中帮其重置，或在此临时修改为一个统一的简单密码，过后再让用户自行修改。因为md5不可能直接转换为bycrypt，而之前已经使用md5保存的密码也不可能通过在.config.php中修改加密方式来改变，所以只能出此下策。

</details>

#### 3. 关于IPv6环境存在的问题
<details><summary>点击展开</summary>

不知从何时起，我的服务器就再也无法使用IPv6来实现前后端对接了。虽然域名解析没有问题，面板也能识别到IPv6地址的访问记录，但XRayR后端就是无法与前端对接。查看log会发现前端返回了“Invalid request IP”的信息，证明面板程序认为后端的IP地址与前端记录的不一致。而事实上，节点IP自动解析为IPv4的情况可以说从远古时期的版本开始就是如此了，而之前即便解析为IPv4，对接上也不会出现什么问题。但这次就变成了例外。

无奈之下，我只能删除了所有域名的IPv6解析，对应的nginx配置文件也将IPv6的监听端口注释掉：
```
server
    {
        listen 80;
        #listen [::]:80;
        server_name www.example.com ;
略
server {
  listen 443 ssl;
  #listen [::]:443 ssl;
```
重新对接前端，果然成功。

事已至此，就我个人而言，只能认为是前端在IPv6的对接上存在问题了。虽说IPv6在节点访问上并不能带来什么明显优势，但我总认为多一个选择总不是坏事。然而有意思的是，我在试图寻找是否有类似案例时，发现SSPanel-Uim的Issue页面也曾经有人提出过这个问题，无情的是，当人问道是不是只有他遇到这个问题时，开发组的回答只有一句“是的，只有你遇到这个问题”。

嗯，多的不想说什么了，我只能再次感慨，要是wiki和注释上啥都有，我也不会写这篇文章了。

</details>

#### 4. Windows及Android客户端的文件包下载
<details><summary>点击展开</summary>

如果大家使用的客户端是clash，那么可以直接执行php xcat ClientDownload，这样在面板里就可以直接下载到Clash了。事实上，当前面板的订阅下发风格也在无限向Clash趋近。但我个人并不太喜欢用Clash，而是使用V2Ray和V2RayNG的方案（当然，iOS似乎只有Clash兼容客户端，例如Shadowrocket），因此上述命令并不能直接完成任务，我们可以手动修改config目录下的clients.json，让它能够自动下载更多的客户端。

找到之前一个客户端对应节的最后一个“{”,并在后面手动加入逗号，然后补充v2rayN和v2rayNG的内容，效果如下：
```
（前文略）  
            ]
        },
        {
        	  "name": "v2rayNG",
            "tagMethod": "github_release",
            "gitRepo": "2dust/v2rayNG",
            "savePath": "public/clients/",
            "downloads": [
                {
                    "sourceName": "v2rayNG_1%tagName1%_arm64-v8a.apk",
                    "saveName": "v2rayNG.apk"
                }
            ]    
        },
        {
        	  "name": "v2rayN",
            "tagMethod": "github_release",
            "gitRepo": "2dust/v2rayN",
            "savePath": "public/clients/",
            "downloads": [
                {
                    "sourceName": "v2rayN-With-Core.zip",
                    "saveName": "v2rayN-With-Core.zip"
                }
            ]
        }
（后文略）
```
保存后执行php xcat ClientDownload即可自动下载最新版本的客户端。

其中，v2rayNG的sourceName有点奇怪，不知道为什么，直接使用%tagName1%会导致执行时版本号识别成“.x.x”，而少了开头的“1”（如1.8.5会被识别成.8.5），导致git提示404下载失败。所以手动添加了个1在前面，让它能够正确下载。如果还是失败，请参照以下步骤手动下载：

先进入网站的public/clients目录，前往[V2RayNG的Release页面](https://github.com/2dust/v2rayNG/releases)，复制apk的下载链接（此处以1.8.5_arm64-v8a为例，不带架构后缀的则为32位、64位全兼容版本），并回到SSH中下载：
```
wget https://github.com/2dust/v2rayNG/releases/download/1.8.5/v2rayNG_1.8.5_arm64-v8a.apk
```
之后修改好客户端文件包的名称：
```
mv v2rayNG_1.8.5_arm64-v8a.apk v2rayNG.apk
```
此时回到面板的“传统订阅”处，检查客户端是否能正常下载。如果报错，可以再次为整个网站目录或这两个文件包设置777权限和root账户，具体参见本节第1点末尾。

</details>

#### 5. Clash及Clash兼容客户端的订阅下发问题
<details><summary>点击展开</summary>

我反正是不敢再向他们提交issue了，谁爱去谁去吧。

在实践中发现，Clash以及Clash兼容客户端，比如Shadowrocket，会出现订阅内容下发不完整的问题。具体表现为缺少path和host的参数，导致节点连接后无法上网。但是使用Xray或v2fly内核的客户端，比如V2RayN和V2RayNG，都可以获得完整订阅内容。

这个问题似乎是伴随着新版面板产生的，同样的客户端在旧版中可以正常使用。

目前个人没有很好的解决办法，只能引导用户手动填写path和host的内容。另外，如果你的server和host内容一致，不填写host似乎也可以正常访问节点。

</details>

#### 三、部署XrayR后端的要点

#### 1. 下载并安装XrayR
<details><summary>点击展开</summary>

执行以下命令，安装XrayR最新版：
```
wget -N https://raw.githubusercontent.com/XrayR-project/XrayR-release/master/install.sh && bash install.sh
```

</details>

#### 2. 节点信息填写以及Custom-config
<details><summary>点击展开</summary>

首先在域名提供商处设置好域名和解析，待生效后，使用lnmp vhost add添加一个网站并设置好SSL认证，过程不再赘述。

完成后，在SSPanel中添加一个节点，填好域名、流量、等级等信息，然后在Custom-config处填写以下代码，对应websocket+tls模式：
```
{
  "offset_port_node": "10086",
  "offset_port_user": "443",
  "server_sub": "www.example.com",
  "host": "www.example.com",
  "alter_id": "0",
  "network": "ws",
  "security": "tls",
  "path": "/welcome/"
}
```
注意，上下两个“{}”符号已经默认存在，所以不需要复制进去。如果你熟悉Vless、Xtls等，请自行修改，本文不做讲解。

其中，port_node为nginx识别到websocket协议后转发给XRayR的端口，也就是XRayR监听的端口。port_user为用户连接到节点时访问的SSL端口（默认为443），server_sub和host一般是相同的，即后端所在服务器的域名。如果你的host不一样，估计你也不需要研究这部分内容了。path是等会儿Nginx配置文件中涉及的内容，可以修改，具体见后文。

编辑好节点信息后，保存节点即可。回到上一页，记住这个节点的ID。

</details>

#### 3. XrayR配置文件
<details><summary>点击展开</summary>

本Repo的“XrayR_config.yml”即本人使用的配置文件模板，将里面的内容直接覆盖掉/etc/XrayR/config.yml中的代码即可，原文件的内容不需要保留。当然，如果你熟悉配置文件的内容，也可以根据自己的需要进行修改。

覆盖好代码后，修改必要的内容即可。通常只需要修改以下内容：
```
ApiHost: "https://www.example.com" #SSPanel面板的网址
       ApiKey: "muKey" #字面意思，你给面板设置的mukey。注意不能使用节点API Key，否则无法对接
       NodeID: 1 #节点ID，把刚才新增节点的ID抄写过来
```

</details>

#### 4. Nginx配置文件
<details><summary>点击展开</summary>

此处有两个步骤。

**可选：如果安装了SSL Preread模块**

此时你可以在nginx.conf中添加stream节来实现同个SSL端口到不同监听端口的分流，也就是V2Ray和Trojan客户端共存（尤其是像soga这样的后端，如果不进行此操作的话，其默认就会监听）。当然你不需要共存也可以进行这步，个人体感是提升了SSL端口的存活率，不容易被封，但实际效果未经验证。

编辑/usr/local/nginx/conf/nginx.conf文件，在events节之后、http节之前添加内容，最终的效果大致如下：
```
events
    {
        use epoll;
        worker_connections 51200;
        multi_accept off;
        accept_mutex off;
    }

stream {
    # map domain to different name
    map $ssl_preread_server_name $backend_name {
    #   www.example.com web;
        v2r.example.com vmess;
    #    trj.example.com trojan;
    # default value for not matching any of above
        default vmess;
    }

#   upstream web {
#       server 127.0.0.1:1442;
#   }

#    upstream trojan {
#        server 127.0.0.1:1443;
#    }

    upstream vmess {
        server 127.0.0.1:1444;
    }

    server {
        listen 443 reuseport;
        #listen [::]:443 reuseport;
        proxy_pass  $backend_name;
        ssl_preread on;
    }
}

http
    {
        include       mime.types;
        default_type  application/octet-stream;
略
```
其中，“v2r.example.com vmess”为“Xray后端使用的域名 转发到的名字”，请首先修改域名。名字对应下方的“upstream vmess”。因此“vmess”等名字你也可以自行修改。listen 443即SSL端口，在不需要跑网站的服务器上，个人建议将它修改成别的端口。注意修改后节点的Custom-config中的port_user需要一并修改。

当然，就算你修改了，也不会影响未经此处转发的网站。例如：在stream这里设置了一个SSL端口为4443，而你又用lnmp vhost add添加了另一个网站并设置SSL，但没有在这里设置stream转发，那么该网站不会受到这边4443端口的影响，依旧按照网站配置文件中的SSL端口（默认443）进行监听。

**必须：修改网站配置文件**

如果你确实不需要做端口转发，则可以直接按照以下内容修改网站配置文件。

假设你的XrayR使用域名为v2r.example.com，则编辑/usr/local/nginx/conf/vhost/v2r.example.com.conf中的内容。将第二个server节（即SSL相关）的内容全部删除，并参照如下内容增加新的server节：
```
server {
  listen 443 ssl; #做了stream转发的，跟nginx.conf中upstream设置的端口号一致。若没有做，则这是SSL端口号。
  #listen [::]:443 ssl;
  ssl_certificate       /usr/local/nginx/conf/ssl/v2r.example.com/fullchain.cer; #修改成自己的域名，下同。
  ssl_certificate_key   /usr/local/nginx/conf/ssl/v2r.example.com/v2r.example.com.key;
  ssl_session_timeout 1d;
  ssl_session_cache shared:MozSSL:10m;
  ssl_session_tickets off;
  
  ssl_protocols         TLSv1.2 TLSv1.3;
  ssl_ciphers           ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384;
  ssl_prefer_server_ciphers off;
  
  server_name           v2r.example.com;
  location /welcome/ { #即节点Custom-config中设置的path，可以修改，确保两边完全一致。
    if ($http_upgrade != "websocket") {
        return 404;
    }
    proxy_redirect off;
    proxy_pass http://127.0.0.1:10086; #即转发给XrayR的端口，对应Custom-config中的port_node。
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
    proxy_set_header Host $host;
    # Show real IP in v2ray access.log
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
  }
}
```
你也可以直接使用本Repo中提供的v2r.example.com.conf中的内容直接覆盖掉原网站配置文件，并修改好相应信息。

修改好后，使用lnmp restart重启LNMP，若没有报错，则修改成功。

</details>

#### 4. 启动节点并查看log
<details><summary>点击展开</summary>

命令行中输入xrayr，打开XrayR主程序，出现如下菜单：
```
  XrayR 后端管理脚本，不适用于docker
--- https://github.com/XrayR-project/XrayR ---
  0. 修改配置
————————————————
  1. 安装 XrayR
  2. 更新 XrayR
  3. 卸载 XrayR
————————————————
  4. 启动 XrayR
  5. 停止 XrayR
  6. 重启 XrayR
  7. 查看 XrayR 状态
  8. 查看 XrayR 日志
————————————————
  9. 设置 XrayR 开机自启
 10. 取消 XrayR 开机自启
————————————————
 11. 一键安装 bbr (最新内核)
 12. 查看 XrayR 版本 
 13. 升级维护脚本
 
XrayR状态: 未运行
是否开机自启: 是

请输入选择 [0-13]: 
```
输入6回车，若之前的设置全部正确，则重启成功。回到主界面，输入8回车，或在终端中输入xrayr log查看日志。

正常情况下，你应该看到类似这样的消息：
```
XrayR 0.9.1 (A Xray backend that supports many panels)
2023/10/22 01:33:15 Start the panel..
2023/10/22 01:33:15 Xray Core Version: 1.8.4
2023/10/22 01:33:15 [Warning] core: Xray 1.8.4 started
2023/10/22 01:33:15 [https://panel.example.com] V2ray(ID=1) Added 10 new users
2023/10/22 01:33:15 [https://panel.example.com] V2ray(ID=1) Start node monitor periodic task
2023/10/22 01:33:15 [https://panel.example.com] V2ray(ID=1) Start user monitor periodic task
2023/10/22 01:33:15 [https://panel.example.com] V2ray(ID=1) Start cert monitor periodic task
```
只要有“Added xx new users”则证明面板中符合本节点条件的用户已经被对接到XrayR后端中，可以正常访问节点。如果出现“Invalid Argument”之类的报错，重点检查XrayR配置文件中的mukey是否正确、面板网址是否正确，“SSpanel”、“V2ray”之类的大小写是否正确（配置文件里有范例，必须严格按照范例的大小写）。

</details>
