---
layout:     post
title:      env_installer:两条命令完成python项目依赖的打包及离线安装
date:       2019-08-27
author:     yandenghong
header-img: img/post-moon.png
catalog: true
tags:
    - open source
    - Python
    - 部署
---
## 前言
本篇旨在介绍自己最近刚刚完成的一个开源项目: [env_installer](https://github.com/yandenghong/env_installer).

因为之前遇到过在内网服务器上部署python项目的情况，发现所有的操作都是可以自动化的，为了提高效率，于是有了[env_installer](https://github.com/yandenghong/env_installer)。该项目是对pip和virtualenv工作流的一层简单封装。

## 正文
pip+virtualenv部署python项目的流程通常是在本地准备好requirements文件,上传至安装了python和virtualenv的服务器后创建虚拟环境并在线安装依赖。
但这在内网服务器上行不通，因为内网无法通过pip下载依赖。而pip也为我们提供了打包的api,将whl格式的依赖包下载至指定目录，安装时再指定查找路径即可
完成离线安装。

本项目是将这个过程中的一些手动操作和原本的打包安装封装到一起，打包只需一条命令，安装也只需一条命令。


首先克隆env_installer到本地(任意路径,最好在你的家目录下)。

假设这是你即将准备部署的项目(位于/home/DemoProject):
```text
DemoProject
├──assets
├──library
├──local
├──settings
├──temp
└──requirements.txt
```

### 打包
```text
usage: pack.py [-h] -b BASE_DIR -r REQ_DIR [-d DOWNLOAD_DIR] [-v]
               [-i INDEX_URL]

arguments:
  -b BASE_DIR, --base-dir BASE_DIR
                        absolute path of the project to be packaged.
  -r REQ_DIR, --req-dir REQ_DIR
                        absolute path of production environment dependency
                        file.
  -d DOWNLOAD_DIR, --download_dir DOWNLOAD_DIR
                        absolute path of package storage location.
  -v                    if need to download an additional `virtualenv`
                        package, use this param. default false.
  -i INDEX_URL, --index-url INDEX_URL
                        download package index url, equivalent to `pip install
                        -i`.

```

此时你的项目的绝对路径是: /home/DemoProject

依赖文件的绝对路径是: /home/DemoProject/requirements.txt

如果服务器上还没有安装virutalenv,则加上`-v`参数，env_installer会为你额外下载它的安装包。

如果默认的pip源下载速度慢，你可以用`-i`更换它，就像`pip install -i` 那样。

到这里，我们的打包命令就完成了: 
```text
python3 pack.py -b /home/DemoProject -r /home/DemoProject/requirements.txt -v
```
这会在你的项目下生成一个目录,里面存储的是按照requirements.txt列出的依赖的whl格式的安装包。

你不必关心这个目录的名字，因为env_installer会处理好一切!

运行成功结果:
```text
Packaged successfully! Saved /home/DemoProject/ea385f6cb21d4097
```

### 安装
```text
usage: install.py [-h] -b BASE_DIR -r REQ_DIR [--venv-pip-path VENV_PIP_PATH]
                  [-p PYTHON]

arguments:
  -b BASE_DIR, --base-dir BASE_DIR
                        absolute path of the project to be deployed.
  -r REQ_DIR, --req-dir REQ_DIR
                        absolute path of production environment dependency
                        file.
  --venv-pip-path VENV_PIP_PATH
                        the pip absolute path of the virtual environment that
                        already exists on the server.
  -p PYTHON, --python PYTHON
                        python interpreter path on the server.equivalent to
                        `virtualenv -p`.

```

现在我们切换到服务器上,确保假设你已经将打包后的项目上传到了服务器的`/deploy/DemoProject`,并且服务器上面已经安装好了python3(`/usr/bin/python3`)。

the absolute path of your project: 
```text
/deploy/DemoProject
```

the absolute path of dependent file: 
```text
/deploy/DemoProject/requirements.txt
```

如果你在打包时使用了`-v`参数，则env_installer认为你要在服务器上安装`virtualenv`,
那么忽略`--venv-pip-path`参数，给出`-p`, python3解释器的路径用于安装好`virtualenv`之后创建虚拟环境。
相反，如果你的服务器上已经准备好了一个virtualenv，那么你只需要给出已经存在的virtualenv下的pip路径即可,例如:`/deploy/ae0719f3_venv/bin/pip3`.

到这里，我们的安装命令就完成了:
```text
python3 install.py -b /deploy/DemoProject -r /deploy/DemoProject/requirements.txt -p /usr/bin/python3
```

运行成功结果:
```text
installed successfully!now you can run:
 source /deploy/DemoProject/6195b07d_venv/bin/activate
to activate the virtualenv.

```
