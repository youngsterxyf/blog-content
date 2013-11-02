Title: 工作中的技术人
Date: 2013-04-23
Author: youngsterxyf
Slug: technical-person
Tags: 工作, 感悟

工作入职半个月，有些事情不太顺利，还没有正式上手工作，也许大公司的节奏便是如此，但我内心是比较急的，希望能尽快地上手做实际的工作，而不是学习和等待。

这半个月里，主要是熟悉工作环境，学习了解工作相关的技术。虽说学习，但其实大部分相关技术以前都了解或使用过，只是经验还不够。

第一周，除了常规的入职事宜，搭建了开发测试环境，并阅读理解工作中使用的web框架。对于这个框架，有太多的吐槽点，严格地说算不上是个框架，可能是因为写得比较早。对于框架，我认为最重要的是为多人协作完成一件事情提供实现上的规范，其次是代码复用，减少工作量。但这个框架除了一些供复用的代码，就啥都没有了。

第二周，学习巩固PHP基础，一直没认真地学习过PHP，只是在实习的时候做了一些开发，稍微了解了下Yii框架和Zend框架，觉得太复杂了点。除此之外，初步了解组内的运维工作，特别是整个系统的架构。


经过一番思考，基于自己的理解，昨天编写了一个玩具性质的MVC web框架原型[minibean](https://github.com/youngsterxyf/minibean)，该框架以路由转发和控制器为核心，所有非静态文件请求的处理都以Application类对象为入口，按照一定规则对请求URI经路由转发找到对应的控制器类，控制器对象中调用模型与视图的类对象等。以后随着开发经验的增加以及对其他开源成熟框架的学习，会不断地完善该框架。在编写该框架的过程中，深感自己经验的不足，特别是对于Model层，以后可能会刻意阅读某些开源框架的Model层实现。

目前组内开发工作还很初步，还没有一个正规的开发流程，也没有明确的开发规范。这样虽然没法从已有的工作中学习很多，但也许有机会参与到这些事情创造过程中，收获会更大。

经过和老同事讨论，以后开发工作涉及的语言和工具包括：Nginx、MySQL、PHP、Redis/Memcached、SVN等，对于这些东西，我都是需要深入学习加强的（当然首先是要解决业务需求）。另外，鉴于原有的那个框架实在不怎么样，以后新的工作可能会选择Zend Framework作为开发框架。

对于工作环境，我觉得不太满意的地方主要是技术氛围不太浓厚，以后有机会和大家一起建立起好的技术氛围，搞搞技术分享讨论什么的。另外，有点憋屈的是，觉着自己被小看了，老同事老觉得应届毕业生啥的不懂，所以也不急着分配具体的工作给我，老让我学习学习再学习。个人认为最好的学习方式是给个具体的需求，具体的问题让我去解决，有经验的同事只需对结果把把关就可以。当我在这过程中遇到搞不定的问题再向他们请教，以这种方式来上手熟悉工作也许更好。我个人也比较喜欢直接丢个实际的问题让我去解决。

对于今后的自己，我有两点忠告：

1.
时刻警惕迷失

虽然工作很重要，要解决业务需求，工作所涉及的技术也应该扎实掌握，深入理解，但不能把自己局限在此，也不能让自己迷失在过于细节的地方。公司提供了一个完善的平台，但这个平台在我看来自足得有些封闭，所以需警惕，要不断地和同事，和公司外面的人交流学习。要经常自我反思，回顾自己走过的路，要让自己的大脑空闲下来花些时间整体规划即将要做的事情。

2.
保持锐气

初步觉着有些同事没什么工作生活技术的激情，可能是长时间工作的缘故，也可能是因为我还太年轻。但目前我还不愿意自己进入那种状态。自己以后应该更加主动积极地对待工作。我喜欢称呼自己为“技术人”，因为“技术人”不仅是搞技术的，更重要的是对技术有热情，有使命感。