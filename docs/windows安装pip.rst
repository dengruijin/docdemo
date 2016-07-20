********************
Windows下安装pip
********************

安装pythton
============
1. 从python官网下载python的windows版安装包，这里用现有的python-2.7.3
#. 配置环境变量
  鼠标右键点击 **我的电脑** --> **属性** --> **高级系统设置**--> **环境变量**。
  在系统环境变量中选择 **Path**进行编辑，在最后面加上 *;C:\Python27*。


安装setuptools
==============
下载https://bootstrap.pypa.io/ez_setup.py ,
下载完后双击 *ez_setup.py* 运行即可。

安装pip
========

1. 下载pip源码包并解压： 网址：https://pypi.python.org/pypi/pip#downloads
#. 在pip源码包解压后的目录中打开CMD，执行 ***python setup.py install***
#. 配置环境变量
  鼠标右键点击 **我的电脑** --> **属性** --> **高级系统设置**--> **环境变量**。
  在系统环境变量中选择 **Path**进行编辑，在最后面加上 *;C:\Python27\Scripts*。


pip install virtualenv
