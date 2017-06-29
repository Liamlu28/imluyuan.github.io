---
layout:     post
title:      "包管理工具setuptools"
subtitle:   "setuptools"
date:       2017-06-28 00:00:00
author:     "Luyuan"
header-img: "img/python/python.jpeg"
catalog: true
tags:
    - Python
---


## 1前言

## 1.1 包管理工具setuptools
setuptools是Python的一个包管理的库，主要对Python程序进行下载、安装、创建、升级和降级。改项目的Github地址https://github.com/pypa/setuptools .当然Openstack中项目同样也是通过Setuptools进行打包安装的，setuptools是Python distutils的增强版本。当你想把一个python程序打包，只需要引入模块中的函数即可。


## 1.2 安装setuptools
```bash
yum install python-setuptools -y
python2-setuptools       noarch                      22.0.5-1.el7                         eos-0.1-openstack-mitaka                      485 k
```

## 1.3 创建一个demo
```
mkdir /tmp/demo && cd /tmp/demo/
```
## 1.4 创建一个setup.py文件
```python
from setuptools import setup, find_packages
setup(
    name = "demo",
    version = "0.1",
    packages = find_packages(),
)
```
## 1.5 执行命令对demo打包
```python
python setup.py bdist_egg

running bdist_egg
running egg_info
creating demo.egg-info
writing demo.egg-info/PKG-INFO
writing top-level names to demo.egg-info/top_level.txt
writing dependency_links to demo.egg-info/dependency_links.txt
writing manifest file 'demo.egg-info/SOURCES.txt'
reading manifest file 'demo.egg-info/SOURCES.txt'
writing manifest file 'demo.egg-info/SOURCES.txt'
installing library code to build/bdist.linux-x86_64/egg
running install_lib
warning: install_lib: 'build/lib' does not exist -- no Python modules to install

creating build
creating build/bdist.linux-x86_64
creating build/bdist.linux-x86_64/egg
creating build/bdist.linux-x86_64/egg/EGG-INFO
copying demo.egg-info/PKG-INFO -> build/bdist.linux-x86_64/egg/EGG-INFO
copying demo.egg-info/SOURCES.txt -> build/bdist.linux-x86_64/egg/EGG-INFO
copying demo.egg-info/dependency_links.txt -> build/bdist.linux-x86_64/egg/EGG-INFO
copying demo.egg-info/top_level.txt -> build/bdist.linux-x86_64/egg/EGG-INFO
zip_safe flag not set; analyzing archive contents...
creating dist
creating 'dist/demo-0.1-py2.7.egg' and adding 'build/bdist.linux-x86_64/egg' to it
removing 'build/bdist.linux-x86_64/egg' (and everything under it)

[root@server-99 demo]# tree
.
├── build
│   └── bdist.linux-x86_64
├── demo.egg-info
│   ├── dependency_links.txt
│   ├── PKG-INFO
│   ├── SOURCES.txt
│   └── top_level.txt
├── dist
│   └── demo-0.1-py2.7.egg   #生成了对应的egg包
└── setup.py

4 directories, 6 files
[root@server-99 dist]# unzip -l demo-0.1-py2.7.egg
Archive:  demo-0.1-py2.7.egg
  Length      Date    Time    Name
---------  ---------- -----   ----
      176  06-29-2017 11:00   EGG-INFO/PKG-INFO
      120  06-29-2017 11:00   EGG-INFO/SOURCES.txt
        1  06-29-2017 11:00   EGG-INFO/dependency_links.txt
        1  06-29-2017 11:00   EGG-INFO/top_level.txt
        1  06-29-2017 11:00   EGG-INFO/zip-safe
---------                     -------
      299                     5 files
```
## 1.6给包添加一个程序
```python
mkdir demo_program  && cd demo_program

touch __init__.py
#!/usr/bin/env python
#-*- coding:utf-8 -*-

def test():
    print "Hello Openstack!"

    if __name__ == '__main__':
        test()

再执行一次命令
python setup.py bdist_egg

[root@server-99 demo]# tree
.
├── build
│   ├── bdist.linux-x86_64
│   └── lib
│       └── demo_program
│           └── __init__.py
├── demo.egg-info
│   ├── dependency_links.txt
│   ├── PKG-INFO
│   ├── SOURCES.txt
│   └── top_level.txt
├── demo_program
│   └── __init__.py
├── dist
│   └── demo-0.1-py2.7.egg
└── setup.py

7 directories, 8 files
多了一个demo_program的目录，那么我来安装一下吧这个小程序
```
## 1.7 安装程序到系统
```python
python setup.py install

Finished processing dependencies for demo==0.1
打开终端我们来测试一下，程序是否可以直接使用
```
## 1.8 测试程序是否可用
```python
>>> import demo
>>> import demo_program
>>> demo_program.test()
    Hello Openstack!
```

—— Luyuan 后记于 2017.06.28
