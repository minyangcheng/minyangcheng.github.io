---
title: window和linux命令操作
date: 2016-11-30 20:08:21
tags: android
---
在弄jenkins给测试机器自动化安装打包好的apk包，学习了下window和linux命令操作，猛然发现其实很多操作都可以通过脚本实现。

<!-- more -->

## window命令

### 脚本编写

直接上一段脚本命令，其中的操作都有注释

```bash
rem 关闭回显
@echo off

rem 设置变量并添加条件判断
set a=abc
set b=bcd 
if %a%==ab (echo one) else if %a%==abc (echo two) else echo three
if %a%==abc (echo four)

rem for运用
for %%i in (a,b,c,d) do @echo %%i
for /r C:\Users\Administrator\Desktop\cmd_dir %%i in (*.*) do @echo %%i

rem choice运用
choice /c ab /n /m "go请选择 a，do go请选择 b。"
if errorlevel 1 (@echo go) else if errorlevel 2 (@echo do go)

rem 等待用户输入
set /p pass=请输入密码:
echo %pass%

rem 删除文件
del /q *123456*

rem 重新开启一个新窗口
start call other.bat

rem 在新窗口中运行jar包
start java -Xmx256m -Xms128m -jar F:\jenkins\workspace\apkRun.jar

rem 窗口暂停
pause
```

### window下非常好用的包管理器Chocolatey

通过chocolaytey安装python和nodejs

```bash
@echo on

pause
echo sure?
pause

rem install chocolatey
@powershell -NoProfile -ExecutionPolicy Bypass -Command "iex ((New-Object System.Net.WebClient).DownloadString('https://chocolatey.org/install.ps1'))" && SET "PATH=%PATH%;%ALLUSERSPROFILE%\chocolatey\bin"

rem install python
choco install python

rem install nodejs
choco install nodejs.install

pause
```

---

## Linux 命令

### 常用的命令

```bash
man ps  //查看命令用法

ps aux | grep tomcat //查看tomcat进程

kill -9 7010  //杀进程

netstat -apn | grep 8099 //查看8099端口有没有被使用

wget http://cn.wordpress.org/wordpress-3.1-zh_CN.zip  //下载文件，默认会以最后一个/后面字符串为名称
wget -O wordpress.zip http://www.centos.bz/download.php?id=1080  //下载文件，并命名为wordpress.zip 

env //查看所有环境变量
echo $JAVA_HOME //查看JAVA_HOME环境变量

locate tomcat//搜索tomcat
```

*定期更新*