---
title: 使用Gooey将命令行转换为GUI界面
date: 2022-06-19 10:19:29 +0800
toc: true
categories: [ python ]
tags: [ python, gooey ]
---

Gooey是一个装饰器库，能将Python命令行程序自动转换为友好的GUI界面程序，对于一些小工具的编写特别有帮助。无需手动编写GUI程序即可快速实现。

官网地址：https://github.com/chriskiehl/Gooey

<!--more-->

## 安装
```
pip install Gooey
```

## 简单示例
这里通过编写一个小工具来演示。工具实现的功能是将某个文件夹下面所有图片找出来后压缩为一个zip文件，然后保存到某个目录下。
那么这个程序的输入是一个文件夹，输出为一个zip文件全路径。

```python
import os
import zipfile
from pathlib import Path
from gooey import Gooey, GooeyParser

@Gooey(program_name='文件夹图片搜集器', language='chinese', encoding='utf-8')
def main():
    parser = GooeyParser(description='将一个文件夹中所有图片打包成一个zip')
    parser.add_argument('Dir', help='文件夹', widget='DirChooser')
    parser.add_argument('Output', help='输出文件名', widget='FileSaver')
    args = parser.parse_args()

    def real_work():
        # 执行耗时操作
        file_list = os.listdir(args.Dir)
        images = []
        for file in file_list:
            extension = Path(file).suffix
            if extension in ['.png', '.jpg', '.jpeg', '.gif', '.svg']:
                images.append(os.path.join(args.Dir, file))

        zip_path = f'{args.Output}'
        with zipfile.ZipFile(zip_path, 'w', zipfile.ZIP_DEFLATED) as zipf:
            for image_path in images:
                zipf.write(image_path, arcname=image_path.split(os.sep)[-1])

        print(f'压缩成功到：{zip_path}')

    real_work()


if __name__ == '__main__':
    main()
```

运行后的效果图

![img.png](https://xnstatic-1253397658.file.myqcloud.com/gooey01.png)

## 打包
通常我们需要将程序打包后分发出去，目前常见的打包工具有pyinsaller和nuitka。这里我使用pyinstaller来演示怎么打包。

### 使用虚拟环境

使用虚拟环境可以减少冗余依赖，强烈推荐。
```
python -m venv pack_env
cd pack_env
# Windows激活命令
Scripts\activate
# 安装最小依赖
pip install pyinstaller python-gooey six
# 安装你的程序依赖（示例）
pip install pandas==2.0.3  # 只安装必要依赖的指定版本
```

### 下载UPX压缩工具（关键步骤）

下载后解压到任意目录（建议路径不要包含中文或空格）：

官网下载：https://github.com/upx/upx/releases

### 准备一个应用图标
下载一个icon格式的图标作为可执行程序的图标。文件名命名为icon.ico，放在python脚本同级目录。

### 打包命令
这个是我最后调试好的单文件打包命令：
```
pyinstaller -F -w \
--upx-dir "D:\libs\upx-5.0.1-win64" \
--upx-exclude "python3.dll"  \
--upx-exclude "vcruntime140.dll"  \
--exclude-module numpy \
--exclude-module panda \
--exclude-module matplotlib \
--exclude-module pytz \
--exclude-module tkinter \
--add-data "D:\Python\Python313\Lib\site-packages\gooey;gooey" \
--add-data "D:\Python\Python313\Lib\site-packages/gooey/languages;gooey/languages" \
--hidden-import=encodings \
--hidden-import "logging" \
--hidden-import "traceback" \
--distpath ./dist \
--workpath ./build \
-i icon.ico \
gooey_zip_pictures.py
```

参数解释：

* -F：生成单个可执行文件
* -w：禁用控制台窗口（适合GUI程序）
* --upx-dir：指定UPX路径（体积压缩关键）
* --exclude-module：排除非必要模块
* --add-data：添加Gooey的依赖资源文件
* -i：设置exe图标（可选）

### 高级优化
还可以在生产的spec文件添加如下内容
```
exe = EXE(...,
    strip=True,  # 去除调试符号
    upx=True,    # 启用UPX
    optimize=2   # Python优化级别
)
```

## 中文编码问题
在打包成单个可执行文件后，执行程序过程中发现界面一直卡住，实际的zip文件已经生成了。最后那句print打印语句报错了。
我把`-w`参数去掉后重新打包后，再次执行能看到控制台窗口的报错信息如下：
```
Exception in thread Thread-1 (_forward_stdout):
Traceback (most recent call last):
  File "threading.py", line 1041, in _bootstrap_inner
  File "threading.py", line 992, in run
  File "gooey\gui\processor.py", line 70, in _forward_stdout
    _progress = self._extract_progress(line)
  File "gooey\gui\processor.py", line 84, in _extract_progress
    find = partial(re.search, string=text.strip().decode(self.encoding))
                                     ~~~~~~~~~~~~~~~~~~~^^^^^^^^^^^^^^^
UnicodeDecodeError: 'utf-8' codec can't decode byte 0xcb in position 2: invalid continuation byte
```

网上查询了各种资料后，最终解决方案如下，在程序开头部分添加编码设置：
```python
import io
import os
import sys
import zipfile
from pathlib import Path

from gooey import Gooey, GooeyParser

# 设置全局UTF-8编码环境
os.environ["PYTHONUTF8"] = "1"

# 处理 sys.stdout 为 None 的情况
if sys.stdout is None:
    # 禁用控制台时，输出重定向到空设备（避免错误）
    sys.stdout = open(os.devnull, 'w', encoding='utf-8')
elif sys.stdout.encoding != 'UTF-8':
    sys.stdout = io.TextIOWrapper(sys.stdout.buffer, encoding='utf-8')


@Gooey(program_name='文件夹图片搜集器', language='chinese', encoding='utf-8')
def main():
    parser = GooeyParser(description='将一个文件夹中所有图片打包成一个zip')
    parser.add_argument('Dir', help='文件夹', widget='DirChooser')
    parser.add_argument('Output', help='输出文件名', widget='FileSaver')
    args = parser.parse_args()

    def real_work():
        # 执行耗时操作
        file_list = os.listdir(args.Dir)
        images = []
        for file in file_list:
            extension = Path(file).suffix
            if extension in ['.png', '.jpg', '.jpeg', '.gif', '.svg']:
                images.append(os.path.join(args.Dir, file))

        zip_path = f'{args.Output}'
        with zipfile.ZipFile(zip_path, 'w', zipfile.ZIP_DEFLATED) as zipf:
            for image_path in images:
                zipf.write(image_path, arcname=image_path.split(os.sep)[-1])

        print(f'压缩成功到：{zip_path}')

    real_work()


if __name__ == '__main__':
    main()

```

