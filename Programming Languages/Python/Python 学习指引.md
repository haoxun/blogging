## pre入门
Python作为一门“好上手”的编程语言，实在太好入门了，如果有C/C++/Java的经验，入门那是分分钟的事情。关于怎么入门的问题，知乎上已经有太多类似的问题了，而且，Python的wiki上还维护了一个[BeginnersGuide页面](https://wiki.python.org/moin/BeginnersGuide)，在此不作详细叙述。

在这个阶段需要达到的目标：

* 代码量1k以上。
* 熟悉常用coding convention，实践过PEP8。
* 熟悉Standrad Library（要求不高，会查文档就行）。


## 入门与深入
到这一步，读者应该对Python有一定程度的了解，至少已经把教材刷过一遍，可以写一点简单的代码，例如在Django框架下写个信息系统什么的。在这个时候就应该对官方的document进行一次深入的学习，查缺补漏，追根溯源。以下是我采用的策略：

1. 快速扫一遍tutorial。如基础扎实可跳过这一步骤。
1. 阅读Language Reference中的[Data model](docs.python.org/3/reference/datamodel.html)。注意了，需要同时阅读Python 2与3的对应部分，以了解2与3的区别。在这一步中需要搞清楚Data Model中的细节，如attribute accessing的机制，special method的含义，descriptor的应用，method creating的场景与过程等。需要注意的是，官方文档中对new-style class的讲解可能有点晦涩，如果出现阅读障碍，
可以尝试阅读[New-style Classes](https://www.python.org/doc/newstyle/)列表中的资料。
1. 相信有了第2步积累，你对Python中的Object已经有了较为深入的了解。这时候你需要继续阅读Language Reference的其余部分，以对Python的语言的细节进行更加深入的学习。
1. 回顾Language Reference，把其中出现的PEP documents扫一遍。

完成上面的步骤，基本上就算是入门Python了，至少在语法层面上不会遇到大的问题，而且遇到问题也至少知道怎么解决问题了。下面就是大量阅读Python的相关资源了，以下是我的建议：

* Python Cookbook（2rd for Py2 and 3rd for Py3）。
* Python Weekly，每周介绍热门项目、讨论与好文章。

于此同时，代码量要提上去。给[PyPI](https://pypi.python.org/pypi)提交你的Package吧！

在这个阶段需要达到的目标：

* 代码量5k以上。
* 熟悉文档，了解所有语言机制。
* 了解与Python相关的常用Package，如unittest等。

## 研究

走到这个阶段的人基本上都知道自己要干什么了，我就不废话了。

