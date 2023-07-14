 # 在LNMP环境下配置SSPanel_Dev与XrayR后端对接的教程
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
[一、安装LNMP的要点](https://github.com/bbs3223474/LNMP-SSPanel_Dev-XrayR-Config/tree/main#%E4%B8%80%E5%AE%89%E8%A3%85lnmp%E7%9A%84%E8%A6%81%E7%82%B9)
   1. Nginx可选模块的选装
   2. PHP必要模块的安装
   3. Redis的安装
   4. MariaDB的安装与升级
   5. 部分Nginx功能的禁用
   6. （可选）低配置VPS下的安装方式

二、部署Dev分支SSPanel-Uim的要点
   
   1. 个人的一些有区别的代码
   2. 旧版数据库迁移的方式
   3. 关于IPv6环境存在的问题

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

- #### 一、安装LNMP的要点
1. Nginx可选模块的安装

总体来说，LNMP本身的安装没有什么难度，但为了后续操作，我们可能还需要修改一些代码。但这一步也不一定每位同学都需要做，你可以根据我下面的说明自行选择。
首先下载LNMP并解压到当前目录：
```
wget http://soft.vpser.net/lnmp/lnmp2.0.tar.gz -O lnmp2.0.tar.gz && tar zxf lnmp2.0.tar.gz && cd lnmp2.0
```

进入lnmp目录后，修改当前目录下的lnmp.conf：
```
vim lnmp.conf
```

分别为Nginx和PHP添加“--with-stream_ssl_preread_module”和“--enable-maintainer-zts”命令。其中，SSL_Preread模块可用于希望V2Ray和Trojan并存、且不会与Nginx监听端口冲突的情况（本文不会涉及，具体原因在附录中解释）。至于Maintainer-Zts，个人在我自己的服务器上并没有启用，暂时不是很清楚这个功能是做什么的，各位可以酌情使用。最终修改的代码如下：
```
Download_Mirror='https://soft.vpser.net'

Nginx_Modules_Options='--with-stream_ssl_preread_module'
PHP_Modules_Options='--enable-maintainer-zts'
略
```

保存退出文本编辑器（vim使用英文冒号+wq回车，FinalShell可以直接Ctrl+S保存，以后不再赘述）。
然后编辑include目录下的main.sh，参照本Repo提供的文件，将1GB内存检测的代码注释掉，避免低配置VPS在编译安装MariaDB等时报错：
```
（第115行）
#    if [ "${Bin}" != "y" ] && [[ "${DBSelect}" =~ ^5|[7-9]|10$ ]] && [ $(free -m | grep Mem | awk '{print  $2}') -le 1024 ]; then
#        echo "Memory less than 1GB, can't install MySQL 8.0 or MairaDB 10.3+!"
#        exit 1
#    fi
略
```

当然，你也可以直接下载本Repo中的文件对它们直接替换，但个人不推荐这样操作。因为LNMP安装包的版本一直在更新，代码也可能会有细微变化，直接替换可能会导致不可预知的后果。

此外，低配置VPS请务必设置好SWAP，即虚拟内存，否则极其容易安装报错。你可以通过swapon -s命令查看当前系统是否存在虚拟内存，如果没有，请参照附录内容手动添加。

以上文件编辑好后，确保当前处在lnmp解压目录下，执行安装脚本：
```
./install.sh lnmp
```

安装时，PHP选择8.1或8.2，MariaDB选择10.11，MemoryAllocator是否安装自行决定，直接开始编译安装即可。作为参考，目前个人使用MariaDB 10.11.2和PHP 8.1.18的组合。
