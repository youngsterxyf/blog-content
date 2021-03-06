---
layout: post
title: 《Classic Shell Scripting》第一、二章阅读笔记
tags: [Linux, Shell]
---

第一章：背景知识
-------------------

软件工具的原则

    一次做好一件事
    
    处理文本行，不要处理二进制数据
    
    使用正则表达式：正则表达式(regular expression)是很强的文本处理机制。了解它的运作模式并加以使用，可适度简化编写命令脚本的工作。
    
    默认使用标准输入/输出：在未明确指定文件名的情况下，程序默认会从它的标准输入读取数据，将数据写到它的标准输出，至于错误信息则会传送到标准错误输出。以这样的方式来编写程序，可以轻松地让它们称为数据过滤器(filter)。
    
    避免喋喋不休
    
    输出格式必须与可接受的输入格式一致：专业的工具程序认为遵循某种格式的输入数据，例如标题行之后接着数据行，或在行上使用某种字段分隔符等，所产生的输出也应遵循与输入一致的规则。这么做的好处是，容易将一个程序的执行结果交给另一个程序处理。
    
    让工具去做困难的部分：虽然UNIX程序并非完全符合你的需求，但是现有的工具或许已经可以为你完成90%的工作。接下来，若有需要，你可以编写一个功能特定的小型程序来完成剩下的工作。
    
    构建特定工具前，先想想：你所要做的事情，是否有其他人也需要做？这个特殊的工作是否有可能是某个一般问题的一个特例？如果是的话，请针对一般问题来编写程序。

第二章：入门
-----------------

\- Shell脚本最常用于系统管理工作，或是用于结合现有的程序以完成小型的，特定的工作。一旦你找出完成工作的方法，可以把用到的命令串在一起，放进一个独立的程序或脚本里，此后只要直接执行该程序便能完成工作。

\- 使用脚本编程语言的好处是，它们多半运行在比编译型语言还高的层级，能够轻易处理文件与目录之类的对象。缺点是：它们的效率通常不如编译型语言。

\- Shell的基本元素：

    1).命令与参数
    2).变量
    3).简单的echo输出
    4).华丽的printf输出
    5).基本的I/O重定向
        (1)重定向与管道
        (2)特殊文件:/dev/null与/dev/tty
    6).基本命令查找

\- 命令行中，分号(;)可用来分隔同一行里的多条命令。Shell会依次执行这些命令。

\- Shell识别三种命令：内建命令，Shell函数以及外部命令：

    1).内建命令就是由Shell本身所执行的命令。有些命令是由于其必要性才内建的，例如cd用来改变目录，read会将来自用户(或文件)的输入数据传给Shell变量。另一种内建命令的存在则是为了效率，其中最典型的就是test命令，编写脚本时会经常用到它。另外还有I/O命令，例如echo与printf。
    2).Shell函数是功能健全的一系列程序代码，以Shell语言写成，它们可以像命令那样引用。 ？？？
    3).外部命令就是由Shell的副本(新的进程)所执行的命令，基本的过程如下：
        a. 建立一个新的进程。此进程即为Shell的一个副本。
        b. 在新的进程里，在PATH变量内所列出的目录中，寻找特定的命令。
        c. 在新的进程里，以所找到的新程序取代执行中的Shell程序并执行。
        d. 程序完成后，最初的Shell会接着从终端读取下一条命令，或执行脚本里的下一条命令。

\- Shell变量名称的开头是一个字母或下划线符号，后面可以接着任意长度的字母，数字或下划线符号。

\- 重定向与管道:

    以<改变标准输入
    以>改变标准输出，>重定向符(redirector)在目的文件不存在时，会新建一个。然而，如果目的文件存在，它就会直接覆盖掉，原本的数据都会丢失
    以>>改变标准输出附加到文件,如同>，如果目的文件不存在，>>重定向符便会新建一个。然而，如果目的文件存在，它不会直接覆盖掉文件，而是将程序产生的数据附加到文件结尾处。
    以|建立管道，管道可以使得执行速度比使用临时文件快上十倍。

\- tr命令

\- 构造管道时，应该试着让每个阶段的数据量变得更少。换句话说，如果你有两个要完成的步骤与先后次序无关，你可以把会让数据量变少的那一个步骤放在管道的前面。这么做可以提升脚本的整体性能，因为UNIX只需要在两个程序间移动少的数据量，每个程序要做的事情也比较少，例如：使用sort排序之前，先以grep找出相关的行，这样可以让sort少做些事。

\- 特殊文件/dev/null与/dev/tty

\- 简单的执行跟踪:

    两种方法：
        1.使用"sh -x scriptname"来执行脚本scriptname；
        2.在脚本中需要跟踪的命令片段前加一句set -x打开执行跟踪的功能，然后在这个命令片段后再用set +x命令关闭它。
