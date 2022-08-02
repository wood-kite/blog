---
layout: post
title:  "Nvidia Jetson Nano部署ROS2 Humble Hawksbill"
date:   2022-06-07 18:05:55 +0800
image:  /2022/jetson-nano-overview-header-bb300-d.jpg
tags:   ros2
---
2022年5月，ROS社区发布了 __ROS2的第一个长期支持版本__ ，也代表着ROS2正式进入了相对稳定可用的状态。        
从前年购入了一个乐视清库存的Astra pro摄像头用于视频聊天开始，知道了ROS，再到去年深入了解ROS，10月份决定要制作一台机器人小车购入Jetson Nano，学习了ROS melodic的基础用法。我就一直在思考：为什么不做一个像区块链一样的“去中心化”的系统，同时又能很好地在系统上运行分布式任务，缓解物联网端设备的运算压力呢？然后我就看到了ROS2。虽然ROS2并没有严格意义上地实现去中心化（ROS2默认的分布式自发现服务，或者可以使用多节点的DDS发现服务避免了ROS1中Master单节点故障导致整个系统不可用）以及像Hadoop集群那样通过分片的方式执行分布式计算，但是在机器人领域，已经覆盖了大量的场景，为大佬们点赞！      
拖拖拉拉地过去了大半年，也没怎么学习ROS，不过也算是把cpp的基础知识学习了，小车的三维建模反复倒腾了三次，最后3D打印完成了车架和外壳的制作和装备工作，终于开始了自己的ROS2学习之路。     
这里推荐一下古月居的教程。      
教程文档：[https://book.guyuehome.com/](https://book.guyuehome.com)       
ROS2入门视频：[https://class.guyuehome.com/detail/p_628f4288e4b01c509ab5bc7a/6](https://class.guyuehome.com/detail/p_628f4288e4b01c509ab5bc7a/6)       
做技术的可能多多少少都会有一些强迫症，而我的强迫症就是“追新”，既然ROS2已经推出了最新的LTS，那还不赶紧用上？虽然实际学习使用foxy版本感觉良好，也没遇到BUG，但顶不住这5年长期稳定支持的香啊，于是乎我开始为我的电脑以及Jetson nano部署ROS2 humble。x86的平台不管是在Ubuntu20.04下的安装包或者编译安装，还是在Ubuntu22.04下的apt、安装包、编译安装都很顺利，但是到了Jetson nano上，就出现了各种自己没法解释的问题，各种搜索工具用尽了，终极大法——源码编译安装也卡在90%的门槛上，具体忘了截图，而且每次重新编译耗时约4.5小时，也就不再截图留念了。      
左思右想，加上看到Ubuntu社区有lxd的解决方案（[https://ubuntu.com/blog/install-ros-2-humble-in-ubuntu-20-04-or-18-04-using-lxd-containers](https://ubuntu.com/blog/install-ros-2-humble-in-ubuntu-20-04-or-18-04-using-lxd-containers)），让低版本（Ubuntu18.04/20.04）甚至是非ubuntu的Linux系统运行ROS2 humble，我决定使用docker来安装该版本，与此同时也能解决ROS2在M1 MacOS安装的问题（我整个编译流程下来，和github上的@kmamykin遇到的编译问题相同：[https://github.com/ros2/ros2/issues/1148](https://github.com/ros2/ros2/issues/1148)，按照@YVbakker的评论，能解决一部分组件问题，但由于不管是foxy还是galactic版本都有大量的组件的代码没有针对darwin的arm64v8处理器做调整，也就没再尝试去逐个攻破，现阶段还是以学习基础的使用和完成规划的目标为主）。        
介绍完以上的问题，来到我们干货时间。        
## 一、Jetson nano 的docker ROS2 humble安装        
### 1.安装docker
```bash
sudo apt-get update & sudo apt-get install -y docker.io
```
### 2.下载ROS2 humle的ARMv8版本镜像
```bash
sudo docker pull arm64v8/ros:humble-perception
```
或（和上面的版本号是一样的，只是提供的名字不同）
```bash
sudo docker pull arm64v8/ros:humble-perception-jammy
```
如果需要下载最小版本，使用：
```bash
sudo docker pull arm64v8/ros:humble-ros-base
```
或（同上版本号）
```bash
sudo docker pull arm64v8/ros:humble-ros-base-jammy
```
或（同上版本号）
```bash
sudo docker pull arm64v8/ros:humble
```
（注：以上所有的版本都没有安装demo_nodes，可以在进入容器后，执行：
```bash
apt-get update & apt-get install -y ros-humble-demo-nodes-cpp ros-humble-demo-nodes-py
```
）
ROS各版本的docker hub的传送门：[https://hub.docker.com/r/osrf/ros2](https://hub.docker.com/r/osrf/ros2)，大家可以在这个上面去找自己想要的版本，ARM64v8专属版本传送门：[https://hub.docker.com/r/arm64v8/ros](https://hub.docker.com/r/arm64v8/ros)。      
### 3.启动docker容器
```bash
sudo docker run -it \
--name ros-humble \
-v /home/bot/dev_ws:/root/dev_ws \
--privileged=true \
--device /dev/spidev0.0:/dev/spidev0.0 \
--device /dev/spidev1.0:/dev/spidev1.0 \
--device /dev/spidev0.1:/dev/spidev0.1 \
--device /dev/spidev1.1:/dev/spidev1.1 \
--network bridge \
arm64v8/ros:humble-perception
```
说明：      
--name 指定容器名称     
-v 指定主机路径到容器内路径的映射，即-v <主机目录>:<容器目录>       
--privileged=true container内的root拥有真正的root权限，主要用于各种io的调用     
--device 指定主机设备到容器内设备的映射，即—device <主机dev>:<容器dev>。这里需要根据具体情况来暴露主机的IO或者设备给docker容器。        
--network 指定docker的网络，已配置的网络可以通过sudo docker network ls来查看。      
arm64v8/ros:humble-perception 镜像名称，已下载的镜像可以通过sudo docker images来查看。      
启动后，会立即进入容器，如果需要开启其他的会话，可以执行以下命令：
```bash
sudo docker exec -it <CONTAINER ID> bash
```
这里的容器ID通过以下命令查看：
```bash
sudo docker images -a
```
### 4.通过这个方法安装的ROS2会有一些小的使用问题，包括前面讲的一些demo没有安装，镜像内没有更新apt安装源以及ros2的apt源没有证书。这个时候需要用到官方文档的方法更新一下ROS2的apt源。     
首先删除原有的ROS2的source.list：
```bash
rm /etc/apt/sources.list.d/ros2-latest.list
```
然后配置证书
```bash
sudo apt update && sudo apt install curl gnupg lsb-release
sudo curl -sSL https://raw.githubusercontent.com/ros/rosdistro/master/ros.key -o /usr/share/keyrings/ros-archive-keyring.gpg
```
最后添加ROS2的apt源
```bash
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/ros-archive-keyring.gpg] http://packages.ros.org/ros2/ubuntu $(source /etc/os-release && echo $UBUNTU_CODENAME) main" | sudo tee /etc/apt/sources.list.d/ros2.list > /dev/null
sudo apt-get update
```
具体的内容可以参考官方的安装说明文档：[https://docs.ros.org/en/humble/Installation/Ubuntu-Install-Debians.html](https://docs.ros.org/en/humble/Installation/Ubuntu-Install-Debians.html)       
通过以上的步骤，就可以在Jetson Nano的最新官方镜像（Ubuntu18.04）上安装单机版的ROS2 humble了，理论上在树莓派等众多基于Arm64v8架构SOC的设备上，只要是Ubuntu18.04以上系统都是可行的，个人推荐使用树莓派，其社区相当活跃，有大量的可以直接移植的代码。（昨天在听古月老师的直播推荐了一款纯国产的开发板：旭日X3派，搭载的是定制的TogetherROS系统，虽然处理器性能上弱于一众树莓派，但其矩阵运算能力强，关键是价格便宜，加上不管是ROS还是ROS2都可以基于网络运行，大型的运算，完全可以让同一局域网内的x86电脑来负担，简直不要太香。）通过以上方法下载的docker镜像，同样能跑在M1 MacOS上，由于MacOS上的docker有别于Linux使用的技术，也就仅限于单机使用，接下来会说明为什么。     
<hr>        

## 二、基于Docker部署的ROS2的集群化        
既然ROS/ROS2是一个基于网络的框架，我前面也提到希望使用到拥有分布式计算能力的系统，那我们这样部署的系统怎么才能实现不同物理节点之间的通信呢？        
不幸的是docker的网络架构不能让我们很方便地暴露端口给外部，必须要通过手动指定-p <主机端口号>:<容器内端口号>来实现对外部端口的暴露，而我们启动的应用出了基础的发现服务有固定的端口段，应用通信的端口在默认情况下是随机的，而且如果把暴露的端口段全部设置好，容器启动后会把未使用的端口也占用掉，导致外部端口资源紧缺，并不是一个非常理想的方式。      
很多云服务商提供了基于容器化技术的虚拟主机服务，这些主机节点之间可以拥有同一个局域网，docker怎么能没有呢？恰好docker提供了docker swarm功能，使得多个服务器或主机上能够创建容器集群服务，组成docker swarm集群组后，通过添加overlay类型的网络，能实现跨物理机的docker集群间通信（同一子网），参考：[https://zhuanlan.zhihu.com/p/384424740](https://zhuanlan.zhihu.com/p/384424740)，本文章的作者在基于docker swarm基础的网络上搭建了一套ElasticSearch的集群（有兴趣的同学可以了解一下弹性搜索以及Apache Lucene全文搜索，ES除了全文搜索，也是一个典型的分布式架构的系统，不过集群间不通过网络自发现，而是配置化的形式来组成集群，而且没有单节点故障问题）。     
### 1.安装ifconfig工具用于查看网络信息
```bash
apt-get update & apt-get install net-tools
```
### 2.查看本地ip地址
```bash
ifconfig
```
![001](/images/2022/220607-001.jpg)     
一般情况下是eth0这个网卡的网络地址，根据设备所在局域网情况确定自己所在的网络ip。        
### 3.创建docker swarm Leader节点
```bash
sudo docker swarm init --advertise-addr=172.29.1.127
```
--advertise-addr是指定节点所在的局域网域，不能使用0.0.0.0来创建。       
执行完成后，系统会反馈一个加入该网络的token，使用命令行提示的脚本，即可让该局域网内的其他计算机加入这个集群。       
![002](/images/2022/220607-002.jpg)     
### 4.加入刚才创建的docker swarm集群        
在同一局域网内的其他计算机上输入加入该集群的命令：
```bash
docker swarm join --token SWMTKN-1-05cszuf3ubnowo0oi5fl6lz379dxckqgguyahpviqzz5usenyl-ayugb6auj3yux1j2cwr5gtgw1 172.29.1.127:2377
```
--token 就是该集群的识别标志        
2377是docker swarm使用的默认的端口号，需要注意防火墙允许该端口通信，避免其他计算机上执行该命令后无法访问当前计算机（Leader节点）的docker swarm服务。        
把需要加入同一网络的计算机都执行以上命令后，可以在Leader节点上查看集群情况，执行：
```bash
sudo docker node ls
```
 ![003](/images/2022/220607-003.jpg)
此时可以看到所有加入的节点，如果有节点下线或者退出了，状态会显示为Down。手动退出集群的方法：
```bash
sudo docker swarm leave
```
### 5.创建用于容器通信的局域网      
在Leader节点上执行：
```bash
sudo docker network create -d overlay –attachable demo
```
-d 指定网络的模式（DRIVER）为overlay（桥接模式和host模式貌似都是直接通过docker0这个虚拟网卡转发到主机所在设备了）。     
--attachable 该参数使网络内的设备可以互相通信。     
demo 局域网络的名称，可以自己随便取，一会儿启动的时候指定的就是这个名称。       
 ![004](/images/2022/220607-004.jpg)        
这里我就不得不吐槽MacOS上的docker了，一开始我发现安装完成后，居然没有docker0这个网桥，我就直接搜刮了一下搜索引擎，发现MacOS上的docker是基于苹果Hypervisor.Framework的虚拟化技术，从FreeBSD的bhyve移植过来的xhyve。而由于其底层的原因（参考[https://breaklimit.me/2017/04/14/mac-osx-vm-1/](https://breaklimit.me/2017/04/14/mac-osx-vm-1/)），无法建立有效的网桥，从而在启动容器的时候无法指定基于overlay的网络，也就没办法和集群中的其他容器进行通信。       
### 6.启动有集群内局域网的容器
```bash
sudo docker run -it \
--name ros-humble \
-v /home/bot/dev_ws:/root/dev_ws \
--privileged=true \
--device /dev/spidev0.0:/dev/spidev0.0 \
--device /dev/spidev1.0:/dev/spidev1.0 \
--device /dev/spidev0.1:/dev/spidev0.1 \
--device /dev/spidev1.1:/dev/spidev1.1 \
--network demo \
arm64v8/ros:humble-perception
```
这一步和前面（一.3）的内容类似，唯一有区别的地方就是—network参数需要指定我们上面创建的基于集群的网络。
同时启动我们配置好的其他计算机的容器。      
### 7.启动ROS2的发现服务        
ROS2有默认的服务发现的功能，但是官方说在WiFi环境下，可能因为多播不稳定的问题，导致无法实现服务自发现（[https://docs.ros.org/en/foxy/Tutorials/Discovery-Server/Discovery-Server.html#background](https://docs.ros.org/en/foxy/Tutorials/Discovery-Server/Discovery-Server.html#background)），我在实际使用中，jetson nano安装foxy版本在WiFi环境下和其他电脑通信没问题，但是放到这次部署的docker容器环境下的humble版本，就不能实现服务自发现，导致在跑demo-nodes-cpp的时候无法顺利通信。我一开始以为docker虚拟的局域网环境是不通的，于是我通过netcat测试了两个容器的udp以及tcp协议的通信都是没有问题的，但是我在listener节点用ros2 topic list查看主题却怎么也看不到talker发布出来的主题。这个时候我才想起了服务发现的问题，于是找到了官方文档中如何启用服务发现（[https://docs.ros.org/en/foxy/Tutorials/Discovery-Server/Discovery-Server.html#run-this-tutorial](https://docs.ros.org/en/foxy/Tutorials/Discovery-Server/Discovery-Server.html#run-this-tutorial)），最终实现了跨容器、跨物理设备的通信。     
引入环境变量
```bash
. /opt/ros/humble/setup.bash
```
启动发现服务
```bash
fastdds discovery --server-id 0
```
--server-id 这里server-id 对应的参数0是指代的发现服务的标识，如果要启动多个发现服务，其他的节点就设置成别的数值（[https://docs.ros.org/en/foxy/Tutorials/Discovery-Server/Discovery-Server.html#server-redundancy](https://docs.ros.org/en/foxy/Tutorials/Discovery-Server/Discovery-Server.html#server-redundancy)）。        
在通信的所有容器中设置发现服务节点
```bash
export ROS_DISCOVERY_SERVER=10.0.2.2:11811
```
这个地方的ip地址（10.0.2.2）需要指定启动发现服务的容器的ip，如果是多个，就设置成如下：
```bash
export ROS_DISCOVERY_SERVER="127.0.0.1:11811;127.0.0.1:11888"
```
启动talker节点，再在非talker的容器内使用ros2 topic list就能看到发布的主题了，同时启动listener也能正常通信了。

