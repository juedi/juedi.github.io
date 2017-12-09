title: Linux文件上传下载工具-lrzsz
date: 2015-03-08 23:17:29
categories: [Linux]
tags: 

---
相信很多使用linux系统的人都少不了向服务器上传、下载文件，以前不怎么了解liunx命令的时候就使用[filezilla](https://filezilla-project.org/)作为windows下的工具。后来发现了一个更加方便的东西：lrzsz。
## 软件安装
### 编译安装
root账号登陆，执行下面的命令：
```bash
wget https://ohse.de/uwe/releases/lrzsz-0.12.20.tar.gz
tar zxvf lrzsz-0.12.20.tar.gz
cd lrzsz-0.12.20
./configure
make && make install
```
上面的命令默认把lsz和lrz安装到了/usr/local/bin/目录下，使用起来不方便，下面创建软连接，并将lsz命名为sz，lrz命名为rz：
```
cd /usr/bin
ln -s /usr/local/bin/lrz rz
ln -s /usr/local/bin/lsz sz
```
### yum安装
```
yum install -y lrzsz
```
## 使用说明
### 下载
```
sz filename
```
记忆方式，sz的s当成send，意味着是服务端发送数据，那在客户端就自然是下载了。
### 上传
```
rz
```
在弹出的文件选择框中选择文件上传。
记忆方式，rz的s当成receive，意味着是服务端接收数据，那在客户端就自然是上传了。