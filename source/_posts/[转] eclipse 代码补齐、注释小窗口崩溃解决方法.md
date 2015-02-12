title: (转) eclipse 代码补齐、注释小窗口崩溃解决方法
date: 2015-01-31 17:34:16
updated: 2015-01-31 17:23:16
categories: [Other]
tags: [other]
---

最近换了新的 ubuntu，eclipse 一出现代码补齐或是注释的小窗口就崩溃了。查看 crash log 文件（在你的 home 目录下）说是 java 虚拟机挂了。网上找了一下，在 eclispe.ini （在 eclipse 文件夹下）中加入一条配置就好了：

<pre config="brush:bash;toolbar:false;">
-Dorg.eclipse.swt.browser.DefaultType=mozilla
</pre>

不过这样设置之后好像注释（javadoc）小窗口的一些 doc 格式好像没用了，哎，算了，总比崩溃强点。


