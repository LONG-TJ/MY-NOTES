# ffmpeg 启用硬件加速

__问题描述__
服务器运行时CPU占用达100%，负载达100%，运行阻塞，通过`top`命令查看发现ffmpeg占用cpu达300%以上。

__解决方案__
通过`ffmpeg -codecs | grep h264`命令查看ffmpeg在服务器上支持的h264编解码器，如图：

<img width="1306" height="346" alt="image" src="https://github.com/user-attachments/assets/1cabd96a-a446-4a2f-98cc-3f89fee89a49" />

通过`ffmpeg -c:v h264_vdpau ..`指定硬件编解码器，解决问题

<img width="950" height="59" alt="image" src="https://github.com/user-attachments/assets/e477cc64-5e4d-452b-be1a-b6471f7b846a" />


