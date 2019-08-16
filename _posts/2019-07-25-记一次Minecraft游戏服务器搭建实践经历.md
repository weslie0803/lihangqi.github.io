---
layout:     post   				    # 使用的布局（不需要改）
title:      记一次Minecraft游戏服务器搭建实践经历 				# 标题 
subtitle:   Minecraft #副标题
date:       2019-07-25 				# 时间
author:     凌洛 						# 作者
header-img: https://oss-weslie.oss-cn-shanghai.aliyuncs.com/data/blog_post_bg/index-hero-og.jpg 	#这篇文章标题背景图片
catalog: true 						# 是否归档
tags:								#标签
    - 游戏服务器
    - Java
---

# Minecraft简介

Minecraft是一款沙盒游戏，整个游戏没有剧情，玩家在游戏中自由建设和破坏，透过像积木一样来对元素进行组合与拼凑，轻而易举的就能制作出小木屋、城堡甚至城市，玩家可以通过自己创造的作品来体验上帝一般的感觉。在这款游戏里，不仅可以单人娱乐，还可以多人联机，玩家也可以安装一些模组来增加游戏趣味性。
Minecraft几乎不包含任何目前流行的游戏元素。Minecraft使用Java编写，具有极强的适应性，而且功能强大，整个游戏画面就像回到了上个世纪，看上去各种模糊和马赛克，就连人物也是一个方块盒子而已，但是却可以给玩家带来像是玩乐高积木一样的永久乐趣。

**需求分析：** 为了使玩家不再孤独地生存在我的世界里，我们可通过服务器搭建游戏联机平台来让我们共同在一个世界里玩耍。

# 配置

下面开始具体的服务器端搭建。

## 服务器选配

本文章采用阿里云云服务器 ** 1 vCPU 1 GB (I/O优化)；ecs.t5-lc1m1.small   5Mbps (峰值)**进行实验配置。若应用于实际的游戏联机场景中，推荐采用“计算网络增强型实例规格族 sn1ne”（更多实例可参考：[云服务器ECS实例规格族](https://help.aliyun.com/document_detail/25378.html?spm=a2c4g.11174283.6.545.32d452feSZyM7m#sn1ne)）。

![配置](https://oss-weslie.oss-cn-shanghai.aliyuncs.com/%E5%9B%BE%E7%89%87/github%E5%8D%9A%E5%AE%A2%E5%9B%BE/b66048d7434571d6e978ab0af112feede7f81075.png)

有关服务器选配可参考：[云服务器ECS](https://help.aliyun.com/product/25365.html?spm=a2c4g.11186623.6.540.259d6a94Kj34Nj)，本文不再赘述。
此服务器可承载约2-5人同时在线。

添加安全组规则：端口范围全部开放。（可参考文档：[添加安全组规则](https://help.aliyun.com/document_detail/25471.html?spm=5176.2020520101.0.d0.460a4df5RL3fuK)）

## 安装JAVA环境
MC是用Java写的就不再赘述了，由于服务器端的MC是一个jar包，我们在配置之后通过运行jar包来开启服务器端，同时我们在PC上打开后通过IP地址即可搜索并进入服务器。所以我们首先要先安装Java。一般来说默认安装的是有Java 8的，如果没有可以按照下面的方法来安装：

**1.验证是否安装Java**，如果安装就查看版本

```shell
java -version
```  

下面是博主的Java版本：

```shell
[root@host ~]# java -version
openjdk version "1.8.0_151"
OpenJDK Runtime Environment (build 1.8.0_151-b12)
OpenJDK 64-Bit Server VM (build 25.151-b12, mixed mode)
```  

如果不是Java 8或者没有安装，就用下面的方法安装：

```Shell
sudo yum install java-1.8.0-openjdk
```  

## 下载Minecraf服务器端

本文章采用Mineraft 1.13 的服务器端版本**，服务器端版本需与客户端版本匹配，否则将会无法连接**。

所以不同的版本对应着不同的服务器端，所以我们要下载正确的版本。如何看MC版本呢，一般来说MC游戏左上角就写的有了，例如：Mineeraft 1.13。如果没有的同学可以启动MC游戏进入到开始界面即可。

![示例1](https://oss-weslie.oss-cn-shanghai.aliyuncs.com/%E5%9B%BE%E7%89%87/github%E5%8D%9A%E5%AE%A2%E5%9B%BE/8730d967ae7a357eb2a0f80c8355df0813fde2d0.png)
输入命令安装1.13版本Minecraft服务器端：

```shell
sudo wget https://launcher.mojang.com/mc/game/1.13/server/d0caafb8438ebd206f99930cfaecfa6c9a13dca0/server.jar
```  

其他服务器端版本可参考：

```shell
sudo wget https://s3.amazonaws.com/Minecraft.Download/versions/版本号/minecraft_server.版本号.jar
```  

## 启动Minecraft
上面的jar包下载完成之后，我们稍作准备就可以开启了！这里还是提醒一下对Java不熟悉的同学。我们知道可执行文件是.exe。双击直接运行，那么Java可以生成jar包，就是.jar文件，在Windows上也是可以双击运行的。而我们下载的这个MC服务器端其实就是已经做好的jar包，所以我们在“双击”启动之前还是需要一些配置，以及用Linux的方法来启动。

首先，来看一下内存的使用，输入命令：

```shell
free -h
```  

我们可以看到当前内存的使用情况，下面是我输入该命令后的显示：

![示例2](https://oss-weslie.oss-cn-shanghai.aliyuncs.com/%E5%9B%BE%E7%89%87/github%E5%8D%9A%E5%AE%A2%E5%9B%BE/11e7333f65f47d85c87f223238d076721e8d54c0.png)
根据free栏的内存数，我们可以确定该给MC服务器分配多大的内存（当然越多越好啦）。
之后，我们就可以使用命令来运行MC服务器了：

```shell
sudo java -Xms[初始启动分配内存] -Xmx[最大分配内存] -jar [jar包所在路径]/minecraft_server.[版本号].jar nogui
```  

以本实践为例：

```shell
java -Xmx666M -Xms1024M -jar server.jar nogui
```  

## 启动成功

上面的命令输入完成，理论上我们就能启动了！这里注意，我们现在如果在Windows下，等于说已经进入到这个程序中了，所以不能再使用Linux命令，等待参数由0%一直到100%就启动完成啦！下面给出博主启动完成的后几行显示：

![示例3](https://oss-weslie.oss-cn-shanghai.aliyuncs.com/%E5%9B%BE%E7%89%87/github%E5%8D%9A%E5%AE%A2%E5%9B%BE/7cb392094e10090394459163d4286d357955815d.png)
这样就启动成功了，不要有顾虑，直接启动PC端连接服务器进入MC吧！（启动失败的情况下面有一个解决方法）

启动完成后当前这个窗口就可以不用管了。博主使用的Xshell，直接最小化。联机完毕之后其实直接关了Xshell就行，自动就断开了，当然也可以**输入Ctrl+C，直接终止进程**。

# 意外状况处理

在进行安装的过程中，并不总是一帆风顺的，往往会出现意外状况。下面给出一些常见的意外状况解决方法。

## 启动失败-同意协议

我们在第一次运行完jar包后，无论是否运行成功，都能发现当前目录下多出了一堆文件，运行失败的时候其实就是配置除了一点问题。我们在当前目录找一下文件：eula.txt。

![示例4](https://oss-weslie.oss-cn-shanghai.aliyuncs.com/%E5%9B%BE%E7%89%87/github%E5%8D%9A%E5%AE%A2%E5%9B%BE/106eca044b88a8567e0d02831f84023e2e075547.png)
我们需要对这个文件进行一下小编辑：

```shell
vi eula.txt
```  

按“i”键进入编辑模式，找到这一行：

```shell
eula=false
```  

将false改成true即可。
退出vi编辑器：
 1.按 esc
2.输入 :wq
这样就同意了那个“最终用户许可协议”。

## PC端连接失败

这里是上面都启动成功之后，PC端也搜索到了服务器，但是就是连接失败，这样我们可以修改配置，先在jar包目录下找到文件server.propertices 并编辑：

```shell
vi server.propertices
```  

找到这一行：

```shell
online-mode:true
```  

将true改为false 即可。

## 游戏配置

关于MC服务器端的配置，我们就需要修改这个文件了，同样在jar包所在目录下：

```shell
vi server.propertices
```  

可以看到，里面是对当前你创建的这个游戏的各种配置，像选择模式啦、世界生成的种子啦、是否有村民啦等等，就像PC创建世界时的各种选项一样。这里就不再介绍了，需要修改的同学自行百度。
# Shell脚本启动
我们如果一直使用上面那一句启动的话是不是非常麻烦！每次都要复制粘贴，那么我们可以写一个简单的Shell脚本，放在jar包所在目录，每次启动的时候直接启动脚本就能进入游戏了，完全不需要Shell编程基础，直接复制粘贴即可！

```shell
vi start.sh
```  

进入编辑模式后输入代码：

```shell
#!/bin/sh
java -Xmx666M -Xms1024M -jar server.jar nogui;
```  

这其实就是让脚本帮你搓启动命令，而你仅需要运行一下脚本即可：

```shell
bash start.sh
```  

最后，在PC端添加Minecraft服务器配置（Server name任意；Server Address填写阿里云服务器的公网IP），并连接。开始享受Mineraft联机的乐趣吧！

![示例5](https://oss-weslie.oss-cn-shanghai.aliyuncs.com/%E5%9B%BE%E7%89%87/github%E5%8D%9A%E5%AE%A2%E5%9B%BE/584bc4f3b8d26fe65153e2f5a7e511022f494228.png)

# 后期运维

可以看到（如下图），选择"1 vCPU 1 GB (I/O优化)；ecs.t5-lc1m1.small   5Mbps (峰值)"的配置，并不满足多人实时在线的流量请求。因此，在选配云服务器中，应选择2 vCPU 4 GB配备5-10Mbps带宽或以上最为合适，可承载20-40人同时在线。

![监控image](https://oss-weslie.oss-cn-shanghai.aliyuncs.com/%E5%9B%BE%E7%89%87/github%E5%8D%9A%E5%AE%A2%E5%9B%BE/eb21eef21e296c724ea0cadb08314a5aae55d8ea.png)
在后期，配备阿里云负载均衡SLB和弹性伸缩等产品，可实现在突发大流量、高并发的场景下的服务支持，使玩家拥有更好的游戏体验。
