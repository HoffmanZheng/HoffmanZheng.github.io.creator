---
title: "多人协作平台 Confluence 搭建"
author: "Chenghao Zheng"
tags: ["tools"]
categories: ["Study notes"]
date: 2023-05-04T13:19:47+01:00
draft: false

---



Confluence 是一个专业的企业知识管理与协同软件，也可以用于构建企业 wiki。使用简单，但它强大的编辑和站点管理特征能够帮助团队成员之间共享信息、文档协作、集体讨论，信息推送。

如果开发环境没有外网， 则可以考虑自行搭建 Confluence 作为产品文档、接口文档、升级日志的协作平台，提高开发协作的效率。本篇采用 Win 10 的 Linux 子系统演示搭建 Confluence 平台，Linux 子系统安装参考：[适用于 Linux 的 Windows 子系统安装指南 (Windows 10)](https://docs.microsoft.com/zh-cn/windows/wsl/install-win10#step-4---download-the-linux-kernel-update-package) 



```shell
// 添加执行权限
chzheng@DESKTOP-RV9C8IN:/usr/src$ chmod +x atlassian-confluence-5.4.4-x64.bin 
chzheng@DESKTOP-RV9C8IN:/usr/src$ ./atlassian-confluence-5.4.4-x64.bin
Unpacking JRE ...
Starting Installer ...
Apr 05, 2021 7:49:11 PM java.util.prefs.FileSystemPreferences$1 run
INFO: Created user preferences directory.
Apr 05, 2021 7:49:11 PM java.util.prefs.FileSystemPreferences$2 run
INFO: Created system preferences directory in java.home.
You do not have administrator rights to this machine and as such, some installation options will not be available. Are you sure you want to continue?
Yes [y, Enter], No [n]
y             // 选项一
This will install Confluence 5.4.4 on your computer.
OK [o, Enter], Cancel [c]
o            // 选项二，是否确认安装
Choose the appropriate installation or upgrade option.
Please choose one of the following:
Express Install (uses default settings) [1], Custom Install (recommended for advanced users) [2, Enter], Upgrade an existing Confluence installation [3]
1            // 选项三，选择安装方式，默认、自定义、升级现有的
See where Confluence will be installed and the settings that will be used.
Installation Directory: /home/chzheng/atlassian/confluence
Home Directory: /home/chzheng/atlassian/application-data/confluence
HTTP Port: 8090
RMI Port: 8000
Install as service: No
Install [i, Enter], Exit [e]
i           // 选项四，确认安装

Extracting files ...


Please wait a few moments while Confluence starts up.
Launching Confluence ...
Installation of Confluence 5.4.4 is complete
Your installation of Confluence 5.4.4 is now ready and can be accessed via
your browser.
Confluence 5.4.4 can be accessed at http://localhost:8090
Finishing installation ...
```



22

~~~shell
chzheng@DESKTOP-RV9C8IN:~/atlassian/confluence/bin$ ./stop-confluence.sh
executing as current user
If you encounter issues starting up Confluence Standalone, please see the Installation guide at http://confluence.atlassian.com/display/DOC/Confluence+Installation+Guide

Server startup logs are located in /home/chzheng/atlassian/confluence/logs/catalina.out
Using CATALINA_BASE:   /home/chzheng/atlassian/confluence
Using CATALINA_HOME:   /home/chzheng/atlassian/confluence
Using CATALINA_TMPDIR: /home/chzheng/atlassian/confluence/temp
Using JRE_HOME:        /home/chzheng/atlassian/confluence/jre/
Using CLASSPATH:       /home/chzheng/atlassian/confluence/bin/bootstrap.jar
Using CATALINA_PID:    /home/chzheng/atlassian/confluence/work/catalina.pid
~~~



confluence5.1-crack

```shell
chzheng@DESKTOP-RV9C8IN:~/atlassian/confluence/confluence/WEB-INF/lib$ cp atlassian-extras-2.4.jar /mnt/c/Users/Chenghao\ Zheng/Downloads/
```





AAABMQ0ODAoPeJxtkMtqwzAQRff6CkHXCn7ElAQEVSxB3foRYqc0S8WdJAJHNpJtmn59lTjZlC5n5
s7hzDxlrcasM9iLcOAtQ38ZhDguK1cEPuJga6O6XrWaxq0+NAPoGlA+nPdgisPWgrGU+Cg2IK8hL
nug103izYkXIbfTy7rP5Rloffo5gT6i2nFmrqlGoL0Z4BESmVQNVXpUVu0beLE1aJjpBolRNsMNT
w+ysTARUuXmFqpLBzd8XGSZ2MQJS5ED6R60dKriu1PmMmmF4TPxAxJEE+BxRNwMtgeTt19gqYdKk
dNdscUZexc4E5jhknG8ZjlnM1SYo9TKTjIq/1ClWqUCV4JlqAQzgkk4XVVvHlnwzSeZR68+SRa7O
brbumma8Ef1v9x6MPVJWvjzy19S54krMC4CFQCByvVcmHBbVsdM0arWAewSX5Vx5QIVAIFG8eb6J
5zTYHjNs6t8rsN4cWzTX02fb





~~~shell
Chenghao Zheng@DESKTOP-RV9C8IN MINGW64 ~/Downloads/soft
$ docker run --name mysql -p 3306:3306 -e MYSQL_ROOT_PASSWORD=123456 -d -v /D/Docker/mysql:/var/lib/mysql mysql:8.0

~~~





create database wiki character set UTF8;

jdbc:mysql://127.0.0.1:3306/wiki?useUnicode=true&characterEncoding=UTF8&sessionVariables=storage_engine%3DInnoDB

