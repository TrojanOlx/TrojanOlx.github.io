---
layout:     post
title:      Python3-rtmp推流,Cv2人脸检测
subtitle:   Python3-rtmp推流,Cv2人脸检测
date:       2019-05-31
author:     Trojan
header-img: img/post-bg-cook.jpg
catalog: true
tags:
    - Python3
---


>环境 python 3.7 ubuntu 18.04


安装依赖
```
pip3 install opencv subprocess
```

```
import cv2.cv2 as cv2
import subprocess as sp

# 视频流服务器地址
rtmpUrl='rtmp://127.0.0.1:1935/live/test1'


# 人脸检测的模型
face_detection =cv2.CascadeClassifier("./trained_models/detection_models/haarcascade_frontalface_default.xml")


# 打开摄像头
camera=cv2.VideoCapture(0)

# 视频属性
size = (int(camera.get(cv2.CAP_PROP_FRAME_WIDTH)), int(camera.get(cv2.CAP_PROP_FRAME_HEIGHT)))
sizeStr = str(size[0]) + 'x' + str(size[1])
fps = camera.get(cv2.CAP_PROP_FPS)  # 30p/self
fps = int(fps)
hz = int(1000.0 / fps)

# 管道输出 ffmpeg推送rtmp 重点 ： 通过管道 共享数据的方式
command = ['ffmpeg',
    '-y',
    '-f', 'rawvideo',
    '-vcodec','rawvideo',
    '-pix_fmt', 'bgr24',
    '-s', sizeStr,
    '-r', str(fps),
    '-i', '-',
    '-c:v', 'libx264',
    '-pix_fmt', 'yuv420p',
    '-preset', 'ultrafast',
    '-f', 'flv',
    rtmpUrl]

pipe = sp.Popen(command, stdin=sp.PIPE)

# 从摄像头中截取每一帧
while True:
    ret, frame = camera.read() # 逐帧采集视频流
    if not ret:
        break

    # 对图片进行人脸检查
    gray_image = cv2.cvtColor(frame, cv2.COLOR_BGR2GRAY)
    rgb_image = cv2.cvtColor(frame, cv2.COLOR_BGR2RGB)

    faces=face_detection.detectMultiScale(frame,1.3,5)

    # 将每个人脸用框框标注出来
    for face_coordinates in faces:
        x, y, width, height = face_coordinates
        cv2.rectangle(frame, (x ,y ), (x + width , y + height), (255,0,0), 2)

    
    pipe.stdin.write(frame.tostring())  # 存入管道

camera.release()
# Release everything if job is finished
#out.release()
print("Over!")

```
