title: Python 使用总结
date: 2015-01-31 17:23:16
categories: [Other]
tags: [other, shell]
---

android 有个自动化的测试工具 monkeyrunner ，是使用 python 编写一段指令，然后自动跑，测试程序。所以就要稍微会点 python 语法啦。

## 不要乱用空格（indent）
平常的其他代码，空格随便用（在不影响美观情况下），但是在 python 中，每行的缩进不要乱用空格。因为空格在 python 中是一个标示符来的。如果你在一行的前面用了几个空格作为缩进符，那么下面的行，你也必须用相同的空格作为缩进符（不能用 tab 代替），否则就会报： Error “mismatched input” expecting DEDENT 这个错误。

虽然可以保证统一用一样的空格，但是还是不要乱用比较好。

## 打印
python 中可以使用 print "xxxx" 进行打印。可以用有些高级的用法，例如 print "a" + "b"（2个必须都是 string）。格式化输出： print "test No.%d" % number


