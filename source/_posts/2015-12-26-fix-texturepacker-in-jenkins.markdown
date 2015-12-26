---
layout: post
title: "解决Texturepacker命令行无法在Jenkins下使用的问题"
date: 2015-12-26 21:11:19 +0800
comments: true
categories: 
---

最近尝试在项目中使用Jenkins，即持续集成。不过在Shell脚本中使用TexturePacker的打包命令时，却无法成功，不管是Mac还是Windows，都会出现同样的提示。

{% codeblock lang:c %}
You must agree to the license agreement before using TexturePacker. Please run the graphical user interface and accept the license terms.
{% endcodeblock %}

奇怪，明明这个agreement，之前在使用TexturePacker GUI时，都已经同意过了的嘛。倒腾了一段时间，才发现，无论是Windows，还是Mac，运行Jenkins的都不是当前的用户。Mac系统下，可以在/Users/Shared中找到一个名为Jenkins的文件夹，Jenkins运行是就用的是这个同名的用户。Windows系统下，可以在《服务》中找到名为Jenkins的服务，右键该服务，选择属性，选择登录这个标签，可以看到运行Jenkins服务的实际上是本地系统账户，而非当前帐号。正因为之前没有在实际运行Jenkins服务的账户下同意过TexturePacker的Agreement，才导致命令行命令无法执行。知道原因后，解决方案就相对简单啦。

Mac: 在terminal中依次使用如下两个命令后，就能看到license agreement弹出来，点击I agree即可。
{% codeblock lang:c %}
1. sudo su jenkins
2. TexturePacker —gui
{% endcodeblock %}

Windows:　如果之前有查看过Jenkins服务的属性，就会发现登录标签页中，不光可以用本地系统账户登录服务，也可以用自己的账户登录Jenkins服务。因此登录自己的账户，同意agreement，TexturePacker的命令行就能用啦。
