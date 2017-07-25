---
title: 使用SecureCRT在远程主机和本地之间传输文件
tags: ["ssh"]
notebook: 工具
---
使用SecureCRT在远程主机和本地之间传输文件
=================
SecureCRT与SshClient不同的就是，SecureCRT没有图形化的文件传输工具，不过也不影响，用命令来实现的话，其实会方便快捷很多。
　　
### 第一种方式：
上传文件只需在shell终端仿真器中输入命令"rz",即可从弹出的对话框中选择本地磁盘上的文件，利用Zmodem上传到服务器当前路径下。

下载文件只需在shell终端仿真器中输入命令"sz 文件名",即可利用Zmodem将文件下载到本地某目录下。

通过"File Transfer"可以修改下载到本地的默认路径。 设置默认目录：options-->session options-->file transfer.

### 第二种方式：用sftp
securecrt 按下ALT+P就开启新的会话 进行ftp操作。
输入：help命令，显示该FTP提供所有的命令
- pwd: 查询linux主机所在目录（也就是远程主机目录）
- lpwd: 查询本地目录（一般指windows上传文件的目录：我们可以通过查看"选项"下拉框中的"会话选项",我们知道本地上传目录为：D:/我的文档）
- ls: 查询连接到当前linux主机所在目录有哪些文件
- lls: 查询当前本地上传目录有哪些文件
- lcd: 改变本地上传目录的路径
- cd: 改变远程上传目录
- get: 将远程目录中文件下载到本地目录
- put: 将本地目录中文件上传到远程主机（linux）
- quit: 断开FTP连接