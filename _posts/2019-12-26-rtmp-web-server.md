---
layout: post
title:  "如何实现网页播放 rtmp 流媒体"
date:   2019-12-26 17:34:00 +0800
image:  
tags:   工作日志
---
最近公司有个需求，要对公司食堂的监控视频进行轻量改造，去除以前对海康插件的依赖，让手机也能顺畅地播放，于是乎在网上搜刮了相关资料，大多数文章都提到了对 rtsp 进行转码处理后，再通过网页的形式播放支持的流媒体，但大多博主充分发挥了 “拿来主义”，却并未进行验证，导致我们在按部就班的时候没法实现我们需要的，在经过了不断地试错之后，现总结出以下的流程，也希望大家能少走弯路。       

监控直播采用了 FFMPEG 转码 ->Nginx 提供 rtmp/hls 服务 ->videojs 网页解码播放的整体思路。        

### 1. 首先需要安装 nginx 以及对应的 nginx 的 rtmp 模块 nginx-rtmp-module-master。      

我们采用的是源码安装，参考自 [https://blog.csdn.net/liuchen1206/article/details/77771703](https://blog.csdn.net/liuchen1206/article/details/77771703)，感谢渔村居士的脚本和软件。下载好该博文提供的脚本软件（可以从这里下载：[http://download.csdn.net/download/liuchen1206/10167705](http://download.csdn.net/download/liuchen1206/10167705)），直接解压后执行脚本就可以实现 ngnix 以及相关组件的安装。软件包括：nginx，nginx-http-flv-module-master，nginx-rtmp-module-master，openssl，pcre，zlib。      

安装完成后，我们配置 nginx，修改 nginx.conf 配置文件，增加 rtmp 设置，设置如下：        
```conf
rtmp {
    server {

        listen 1935;
        chunk_size 4000;
        max_streams 1024;
        ack_window 5000000;

        application live {
            live on;
            exec_options on;
            publish_notify on;
            hls on;
            wait_video on;
            wait_key on;
            record off;
            record_path html/record;
            hls_path html/hls;
            hls_fragment 3s;
        }
    }
}
``` 
在我们的配置中，开启了 hls，因为我们页面是使用的 hls 的方式播放。其中：     
hls_path 为 hls 分片文件存放的目录，我们这个地方配置的是相对路径；      
hls_fragment 为分片文件的视频时长，这里为 3s，网页通过 m3u8 加载了分片文件信息列表，再根据文件信息顺次加载这些视频文件，所以理论上分片文件越小，我们感受到的延时会越低，当然还会受到转码、网络延迟等因素的影响，我们在测试阶段的时候，在设置 3s 的情况下，延迟有时候甚至会达到 2 分钟，甚至是去加载已经过期的分片文件导致视频没有正常播放，这个在后期需要进行相应的优化。       
hls_playlist_length：该参数我们在这里没有设置，是默认的 30s，可以适当将该参数设置长一点，避免网络延时大的时候导致前端页面在加载过去时间段的视频找不到文件的问题。       
其余的参数可以参考 [https://www.cnblogs.com/tinywan/p/5981197.html](https://www.cnblogs.com/tinywan/p/5981197.html)     
修改完配置文件后，我们就可以启动 nginx 了，启动后，除了默认的 80 端口，如果设置了 https 的会有 443 端口等等，除了确认以上的端口信息，还会有我们配置中的 rtmp 的 1935 端口，如果没有该端口，需要再次检查配置文件。       

![001](/images/2019/191226-001.webp)        

### 2. 安装 ffmpeg
我们是在 centos 环境下通过 yum 源进行安装的，大家可以参考网上的安装方法。       
首先添加 yum 源，我们这里是 centos7 的 yum 源：     
```bash     
yum install -y epel-release
rpm --import http://li.nux.ro/download/nux/RPM-GPG-KEY-nux.ro
rpm -Uvh http://li.nux.ro/download/nux/dextop/el7/x86_64/nux-dextop-release-0-5.el7.nux.noarch.rpm
yum install -y ffmpeg
```     
（可以参考 [https://www.cnblogs.com/zepc007/p/11123921.html](https://www.cnblogs.com/zepc007/p/11123921.html)）     
安装完毕后，就可以启动转码了。执行如下命令      
```bash     
ffmpeg -re  -rtsp_transport tcp -i "rtsp://127.0.0.1" -f flv -vcodec libx264 -vprofile baseline -acodec aac -ar 44100 -strict -2 -ac 1 -f flv -s 1280x720 -q 10 "rtmp://127.0.0.1:1935/live/test"
```     
其中 - i 参数后跟的是视频源，即摄像头的拉流地址，最后不带参数的地址（rtmp://127.0.0.1:1935/live/test）为推流地址，可以看到，推流地址的端口为 1935，就是 nginx 的服务地址，后面的路径 /live/test 中 live 为配置的 application live，test 可以随便指定，test 名称决定了服务的具体名称，生成的 m3u8 文件会以此名称命名。       
![002](/images/2019/191226-002.webp)        

提交后就会开始初始化参数：      
![003](/images/2019/191226-003.webp)        

初始化完毕后就会启动转码        
![004](/images/2019/191226-004.webp)        
Invalid UE golomb code 和 error while decoding MB 71 63, bytestream -5 是由于某些配置问题，可以忽略，不会影响实际效果，具体我们在此处没有处理，实际上线需要处理这个问题，相关的可以在网上搜索到，后续我会在这儿更新具体的调整方法。     
此时，后台的服务已全部配置完毕，后续的就是展示的页面，代码如下：        
```html     
<!DOCTYPE html>
<html>
<head>
  <link href="https://cdn.bootcss.com/video.js/7.6.5/video-js.css" rel="stylesheet" />
  <script src="https://cdn.bootcss.com/video.js/7.6.5/video.min.js"></script>
  <script src="https://cdn.bootcss.com/videojs-contrib-hls/5.15.0/videojs-contrib-hls.js"></script>
</head>
    <body>
        <section id="videoPlayer">
            <video id="example-video" width="600" height="300" class="video-js vjs-default-skin vjs-big-play-centered" poster="">
                <source src="http://172.0.0.1/hls/test.m3u8" type="application/x-mpegURL" id="target">
            </video>
        </section>
        <script type="text/javascript">
            var player = videojs('example-video', { "poster": "", "controls": "true" }, function() {
                this.on('play', function() {
                    console.log('正在播放');
                });
                //暂停--播放完毕后也会暂停
                this.on('pause', function() {
                    console.log("暂停中")
                });
                // 结束
                this.on('ended', function() {
                    console.log('结束');
                })
            });
        </script>
    </body>
</html>
```     
只需要替换相应的 m3u8 即可实现资源播放，这里采用的是 video.js 的方案。      
![005](/images/2019/191226-005.webp)        
![006](/images/2019/191226-006.webp)        
![007](/images/2019/191226-007.webp)        

m3u8 文件中会有播放列表信息     
![008](/images/2019/191226-008.webp)    


