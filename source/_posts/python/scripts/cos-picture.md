---
title: 腾讯云COS上传图片
date: 2023-05-09 11:23:15 +0800
toc: true
categories: [ python ]
tags: [ python ]
---

写博客或者笔记的时候，会经常要将图片上传至腾讯云对象存储COS上面，需要用QQ登录腾讯云再手动上传，稍显麻烦。
看了下腾讯云开放API，对接很简单。

首先创建一个子账号，然后在`访问管理-访问密钥-API密钥管理`里面创建一对新密钥SecretId/SecretKey。复制保存下来。

COS有很多语言实现的SDK，Python版本的地址：<https://cloud.tencent.com/document/product/436/12269>

首先pip安装：

```bash
pip install -U cos-python-sdk-v5
```
<!-- more -->

然后就可以直接上代码了：

```python
# -*- encoding: utf-8 -*-
"""
使用腾讯云COS的SDK来上次图片
appid 已在配置中移除,请在参数 Bucket 中带上 appid。Bucket 由 BucketName-APPID 组成
设置用户配置, 包括 secretId，secretKey 以及 Region
"""
import logging
import os
import shutil
import sys

from qcloud_cos import CosConfig
from qcloud_cos import CosS3Client

logging.basicConfig(level=logging.INFO, stream=sys.stdout)

secret_id = 'xxxxx'  # 替换为用户的 secretId
secret_key = 'xxxxx'  # 替换为用户的 secretKey
region = 'ap-shanghai'  # 替换为用户的 Region
token = None  # 使用临时密钥需要传入 Token，默认为空，可不填
config = CosConfig(Region=region, SecretId=secret_id, SecretKey=secret_key, Token=token)
# 2. 获取客户端对象
client = CosS3Client(config)


def is_image(file):
    return file.endswith('.jpg') \
           or file.endswith('.jpeg') \
           or file.endswith('.png') \
           or file.endswith('.ico')


def upload_image(image):
    # 高级上传接口(推荐)
    response = client.upload_file(
        Bucket='xnstatic-1253397658',
        LocalFilePath=image,
        Key=image,
        PartSize=10,
        MAXThread=10
    )
    print(response['ETag'])


# 先创建备份文件夹
backup = os.path.join(os.curdir, 'backup')
if not os.path.isdir(backup):
    os.mkdir(backup)

files = [f for f in os.listdir(os.curdir) if os.path.isfile(f) and is_image(f)]
for image in files:
    upload_image(image)
    shutil.move(os.path.join(os.curdir, image), os.path.join(backup, image))
```

官方的文件上传下载例子：
<https://github.com/tencentyun/cos-python-sdk-v5/blob/master/qcloud_cos/demo.py>

最后，如果在windows上面开发，写个bat脚本，双击即可运行：

```batch
@echo off
REM 声明采用UTF-8编码
chcp 65001
cd %~dp0
echo 当前目录：%~dp0
python cos_image.py
pause
```

把这两个脚本文件放在某个文件夹目录下面，每次想上传图片了就复制到这个文件夹，双击bat搞定。
