---
permalink: /
title: "个人简历"
author_profile: true
redirect_from: 
  - /about/
  - /about.html
---
{% include archive-single.html %}
---
## 基本信息
- 姓名：甘劲搏
- 院校：湖北工业大学
- 邮箱：gjb2574562636@outlook.com<br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;
duxingmengshou@gmail.com

## 教育背景

**2021.09 - 至今** 湖北工业大学 计算机科学与技术（本科）
- 专业成绩：GPA：3.9614 / 5.0；专业排名：3 / 148
- 主修课程：
    - 面向对象程序设计(C++)：100
    - 数据结构与算法基础：99
    - 互联网程序设计：98
    - 数字图像处理：94
    - 算法分析与设计：93
    - 数据库原理及应用：93
    - 嵌入式原理与接口技术：93
    - 数字逻辑：92
- 获奖经历：
    - 2023 年全国大学生物联网设计竞赛（华为杯）华中及西南赛区一等奖 （2023）
    - 第十七届 iCAN 大学生创新创业大赛省三等奖 （2023）
    - 第九届“互联网 +”大学生创新创业大赛校赛二等奖 （2023）
    - 校三等奖学金、校二等奖学金、国家励志奖学金（2021 - 2023）
- 其他：
    - 省级大学生创新创业训练项目（S202310500081），深度视觉感知编码集成融合的多视图病理切片柔性配准与分析平台

## 技术技能
- 熟悉C++新标准（C++17）、STL库（容器、算法、thread、memory）、Boost库（asio）、QT框架、OpenCV、gRPC、MysqlConnector、Hiredis
- 熟练使用 Python 构建脚本程序；熟悉 Flask、Django 等后端框架、OpenCV-Python。
- 熟悉Linux命令，shell脚本编写，对 Linux 源码有一定了解，进行过Uboot+Linux内核裁剪的移植。
- 熟悉 Oracle、MySQL、SQL Server 等数据库使用。
- 掌握 Git、Vim、Navicat、Docker、cmake、makefile的使用
- 熟悉各种网络协议（TPC、UDP、gRPC等），多线程异步开发，对图像处理，内核驱动开发有所了解。

## 项目经验

### 暮光之眼 - 智能家居监控（互联网 +） - 下位机 Linux、后端、前端
- **项目描述**：使用计算机视觉识别技术实现对独居老人的实时监控，平台检测到异常行为（如老人跌倒、突发疾病等）时，自动向家属汇报，提供远程提醒等功能服务。
- **使用技术**：Flask、Vue.js、OpenCV、MySQL、YOLO
- **责任描述**：
    - 下位机使用 OpenCV 获取摄像头视频，使用 YOLOv5 进行异常行为检测，构建 TCP 客户端连接至服务端。后改为使用 ESP32 + FreeRTOS 进行视频上传，服务器进行推理。
    - 构建 TCP 服务器进行视频的接收以及转发（转MJPEG流）。
    - 使用 Flask + MySQL 构建后端服务，以支持多摄像头加入以及检测参数传递等。
    - 前端使用 Vue 进行数据可视化（ECharts）和实时监控（视频监控、log、危险提示）。

---

### 智能电梯监控平台（全国大学生物联网设计竞赛） - 下位机 Linux、后端、前端
- **项目描述**：使用目标检测算法在电梯端对电动车或者其他危险品进行检测拦截，同时对单独乘用电梯的小孩进行拦截。利用华为云物联网平台进行实时的传感器信息上传，实现多小区多电梯监控提醒。利用 three.js 进行场景 3D 建模（数字孪生）。
- **使用技术**：YOLO、Django、华为云物联网平台、Vue.js、MySQL、OpenCV
- **责任描述**：
    - 设备端使用 OpenCV 获取摄像头视频并检测（YOLOv8），使用std::thread进行多核负载均衡。
    - 使用毫米波传感器、超声波传感器进行儿童判断以及人员异常行为判断（Python），使用MQTT协议上传云物联网平台。
    - 使用 Django 搭建后端服务接收传感器信息和RTSP推流视频，前端使用 Vue.js + ECharts 进行数据可视化。

---

### 双目视觉识别与深度定位 - 全栈开发

- **项目描述**：使用双目相机进行的识别与深度定位，识别功能同上，基于双目摄像头厂家提供的驱动进行进一步开发（识别与定位融合）。
- **主要工作**：厂家只提供了基本深度图获取的源码，实际要求上传识别并结合深度图进行30m范围内的识别与定位，由于深度图属于与原图共生的一种附加信息，不能简单使用FFmpeg等进行推流，使用TCP进行图像信息的传递，服务端实现图像获取、信息融合检测。

---

### SocketMjpegStreamer

- **项目描述**：基于上面项目抽象并实现的一个中间层，实现TCP/UDP的视频或者其他信息的输入（自定义消息格式）并转发为HTTP视频流或者其他信息（基本HTTP服务器），支持多客户端的TCP/UDP输入，多客户端的HTTP输出（自实现线程池 ）。
- **主要工作**：
  - 使用Python的Twisted库+OpenCV进行原型开发。
  - 使用C++（Boost+多线程）进行重写优化，自实现线程池。
  - 对于高并发的请求以及日志写入，自实现异步日志库（1443 w条log/s，单条日志100B）
- **项目地址**：https://github.com/Duxingmengshou/PySocketMjpegStreamer；https://github.com/Duxingmengshou/BooAsyncLog；https://github.com/Duxingmengshou/boost-mjpeg-streamer

---

### 嵌入式人脸识别 - 企业实习

- **项目描述**：移植Linux、获取摄像头视频、完成按钮以及指示灯驱动、QT多页面LCD屏显示、UDP上传人脸检测、Python人脸视频服务器。
- **主要工作**：
  - 在移植的Linux中实现基本驱动以及OpenSSH等基本软件的移植。
  - 基于上述实现的驱动进行多线程QT开发，多页面切换，指示灯显示，按钮操作。
  - 异步UDP上传人脸信息后台匹配，使用Dlib进行人脸识别检测。
- **项目地址**：https://github.com/Duxingmengshou/EmbeddedFaceRecognition

---

### gRPC分布式网络聊天器

- **任务描述**：使用gRPC、Boost、jsoncpp、QT实现的全栈分布式网络聊天器，数据库使用MySQL、用Redis做缓存，使用cmake进行项目管理。