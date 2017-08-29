---
title: 使用jenkins搭建持续集成环境
date: 2016-11-13 12:21:08
tags: tool
---
jenkins是一个自动化引擎，它提供了许多插件，让你能够持续集成、自动化测试和持续交付。本文所讨论的是在window系统上搭建jenkins环境，并新建一个构建android apk文件的任务。

<!-- more -->

## jenkins 环境搭建

### 下载jenkins安装包并安装

从官网下载安装包，双击安装即可，安装过程中根据界面上的提示进行操作。因为安装中一个步骤，会提示你选择下载jenkins插件的，所以最好fq进行安装，这样会快一点。最终你会得到一个用户名和密码，后面需要用到。

> jenkins官网：<https://jenkins.io/index.html>

### 启动和关闭jenkins

安装完毕后，jenkins服务是默认启动的，你也可以手动启动和关闭。
```bash
//关闭jenkins服务
net stop jenkins
//开启jenkins服务
net start jenkins
```

启动后可以在自己机器上访问http://127.0.0.1:8080/进入jenkins首页。

### 配置jenkins参数

1. 用已知的用户名和密码登陆jenkins。
2. 进入系统设置界面，添加编译android项目所需要的sdk环境变量
![](/images/jenkins_1.jpg)
3. 进入Global Tool Configuration界面，分别添加jdk、git、gradle环境变量，其他的环境变量，你可以根据需求添加即可。
![](/images/jenkins_2.jpg)
![](/images/jenkins_3.jpg)

## 新建任务

1. 点击<新建>，选择<构建一个自由风格的软件项目>。
2. general面板：
![](/images/jenkins_4.jpg)
3. 源码管理：
![](/images/jenkins_5.jpg)
添加Repository URL时会让你选择Credentials，如果没有则需要add URL时会让你选择Credentials。
git仓库类型的Credentials的添加：
![](/images/jenkins_8.jpg)
svn仓库类型的Credentials的添加：
![](/images/jenkins_9.jpg)
4. 构建:
![](/images/jenkins_6.jpg)
5. 构建后操作:
![](/images/jenkins_7.jpg)

## 启动任务

在jenkins首页点击任务名称，即可进入任务详情界面，点击<立即构建>即可开始构建apk文件，点击Build History中的条目即可进入构建详情。
![](/images/jenkins_10.jpg)

## 添加节点

添加节点可以让你的任务在节点所在的机器上运行，你可以分布式的去完成一项任务。

1. 在Configure Global Security中将TCP port for JNLP agents指定为随机选取
![](/images/jenkins_11.jpg)
2. 在系统设置中指定jenkins所在机器的ip和端口
![](/images/jenkins_12.jpg)
3. 添加节点
![](/images/jenkins_13.jpg)
4. 点击Launch图标下载jnlp文件，并放入远程工作目录下
![](/images/jenkins_14.jpg)
5. 进入window控制面板，选择java，进入java控制面板，选择安全，将jenkins的服务器ip地址添加到“例外站点列表”
![](/images/jenkins_15.jpg)
6. 在远程工作目录下，右键jnlp文件以java方式运行。

## 用jenkins实现一个有趣的功能

想象一下：当bug修复完毕后，开发人员将代码上传到git或svn，然后告知测试人员，可以进行测试。测试人员知道后，便可以在jenkins上立即构建相应项目，当项目构建好后，又会以测试的电脑为节点，去给测试弹出一个窗口，让测试去选择需要安装刚刚编译好的apk包，并且直接给测试人员的测试手机安装。

实现这样一个个其实用jenkins时非常简单的。主要的难点在与如何在节点机器上弹出命令行窗口。我的方法是：首先用java se写一个可供用户选择安装的命令界面，即为可运行的jar包，然后再jenkins中添加节点，并在节点上用bash命令行去执行这个jar包即可。

在任务构建完毕后，在节点机器上执行的bash
```bash
//启动jar包，在节点机器上打开命令行窗口
start java -Xmx256m -Xms128m -jar F:\jenkins\workspace\apkRun.jar
```

可执行jar包内部的代码
```java
package com.min;

import java.io.BufferedReader;
import java.io.File;
import java.io.InputStream;
import java.io.InputStreamReader;
import java.util.HashMap;
import java.util.Iterator;
import java.util.Map;

public class App {
	
	//执行语句:start /i java -jar C:\Users\cheguo\Desktop\ApkRun.jar
	public static void main(String args[]){
        try {
        	File apkDir=new File("F:\\jenkins\\workspace\\machine_install_tuoche\\app\\apks");
    		if(!apkDir.exists()||apkDir.listFiles().length==0){
    			System.out.println("请现在jenkins平台上点击立即构建或构建出问题了");
    			return ;
    		}
    		File[] apkFileArr=apkDir.listFiles();
    		Map<Integer,File> fileMap=new HashMap<>();
    		for(int i=0;i<apkFileArr.length;i++){
    			fileMap.put(i+1, apkFileArr[i]);
    		}
    		
    		//输出提示选项
    		System.out.println("===================输入选项===================");
    		Iterator<Integer> iterator=fileMap.keySet().iterator();
    		while(iterator.hasNext()){
    			Integer key=iterator.next();
    			File value=fileMap.get(key);
    			System.out.println(value+"   <---->   "+key);;
    		}
        	
            System.out.print("请输入安装包后面的选项:");  
            BufferedReader br = new BufferedReader(new InputStreamReader(System.in));
            String s=null;
            int inputKey=-1;
            while((s= br.readLine())!=null){
            	try {
					inputKey=Integer.parseInt(s);
				} catch (Exception e) {
				}
            	if(fileMap.get(inputKey)!=null){
            		break;
            	}else{
            		System.out.println("请输入争正确的选项值");
            	}
            }
            File selectApkFile=fileMap.get(inputKey);
            System.out.println("您选择安装的包为:"+selectApkFile.getName());
            
            System.out.println("===========开始安装==================");
            String cmdStr="adb install -r "+selectApkFile.getAbsolutePath();
            Process process=Runtime.getRuntime().exec(cmdStr);
            InputStream is=process.getInputStream();
            BufferedReader infoReader=new BufferedReader(new InputStreamReader(process.getInputStream()));
            while((s=infoReader.readLine())!=null){
            	System.out.println("安装进度----"+s);
            }
            infoReader.close();
            System.out.println("安装完成");
            
            //停住界面
            System.out.println("按q键退出");
            br = new BufferedReader(new InputStreamReader(System.in));
            s=null;
            while((s= br.readLine())!=null){
            	if(s.equals("q")){
            		break;
            	}
            }
            br.close();
        } catch (Exception e) {
            e.printStackTrace();  
        }  
	}

}
```