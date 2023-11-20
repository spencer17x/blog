---
title: Miniconda使用教程
date: 2023-10-12 15:46:35
tags:
  - Others
---

## Miniconda简介

Miniconda是一款小巧的python环境管理工具，安装包大约只有50M多点，其安装程序中包含conda软件包管理器和Python。一旦安装了Miniconda，就可以使用conda命令安装任何其他软件工具包并创建环境等。

## 下载

https://docs.conda.io/projects/miniconda/en/latest/

## 常用命令

**1.查看所有工具包**

```shell
conda list
```

**2.查看当前存在哪些虚拟环境**
```shell
conda env list 
conda info -e
```

**3.检查更新当前conda**
```shell
conda update conda
```

**4.Python创建虚拟环境**
```shell
conda create -n [env_name] python=x.x
或者克隆
conda create -n your_name --clone env_name
```

**5.激活虚拟环境**
```shell
conda activate [env_name]
```

**6.对虚拟环境中安装额外的包**
```shell
conda install -n env_name [package]  # 未激活环境
conda install [package]  # 如果已经激活环境
```

**7.关闭虚拟环境(即从当前环境退出返回使用PATH环境中的默认python版本)**
```shell
source deactivate  
conda deactivate 
```

**8.删除虚拟环境**
```shell
conda remove -n [env_name] --all
```

**9.删除环境中的某个包**
```shell
conda remove -n [env_name] [package]
```

**10.设置国内镜像**
```shell
conda config --add channels https://mirrors.tuna.tsinghua.edu.cn/anaconda/cloud/msys2/
conda config --add channels https://mirrors.tuna.tsinghua.edu.cn/anaconda/cloud/conda-forge/
conda config --add channels https://mirrors.tuna.tsinghua.edu.cn/anaconda/pkgs/free/
conda config --set show_channel_urls yes
```

**11.恢复默认镜像**
```shell
conda config --remove-key channels
```

**12.查看当前镜像**
```shell
conda config --show channels
```

**13.安装某些包**
```shell
conda install -c anaconda scikit-learn    # 安装sklearn
pip install -i pypi.douban.com/simple tensorflow-gpu==1.14   #用豆瓣源安装包，上面的清华园同理，记得-i
```

