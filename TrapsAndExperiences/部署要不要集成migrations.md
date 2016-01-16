#### 两种ansible部署方式:

    1. 把migrations文件放进git里， 部署的时候先把目标环境的migrations文件夹清空，然后同步最新的文件夹过来，再执行migrate操作。
    2. migrations文件不进行git管理，每一次部署也不删除目标环境的migrations文件，直接跑makemigrations， migrate。

----

对于方式一（也是目前公司的做法），有一个明显的好处就是因为migrations文件纳入git里，对于版本管理是十分有利的，保存了migrations文件就相当于保存了model的版本历史，可以很轻易地查看model的变更。如果我们在一段代码里使用了某个model的属性或方法，几个版本过去了，这段代码一直没有被访问到，而对应的model已经己经修改。在最近一次版本中终于访问了，然后报错。这时就可以很方便地根据model的migrations文件来修复这些兼容问题。

但麻烦的地方在于，migrations加入git的时机，比如一个项目是多人开发的，有可能会对一个model进行重复修改，或者对不同的model修改。等到合并分支的时候就会发现，a分支的migrations是基于主分支的最后一个migrations文件的，而b分支也是，如此一来migrations的依赖链就是非线性的了（出现了类似并联的两个文件），虽然django自带有merge的选项可以自动合并migrations，但并不一定成功。不成功时就需要手动改写migrations，相当繁琐。当然另外一种方法是，分支开发的提交不包含migrations文件，而是合并到主分支之后（model的结构固定下来），再在主分支上运行makemigrations，这样可以确保migrations依赖链始终都是线性的，不会出现冲突。

方式二对于方式一的优点自然是免去merge migrations的烦恼，也不用每次主分支model有更改后执行makemigrations。他的migrations是基于部署的，比方说一个项目最初上线的版本是0.1, 执行了初次migrations，因此各种app model的migration文件应该都是 0001_initial.py这样。等到项目更新了一些功能，再升级到0.2版本，按照方式二，在部署的时候执行migrations， 必然是基于0001_initial的，因此有变动的models最终会产生0002_xxx.py这样的migration文件。而相应的如果用方式一部署，在版本0.1到0.2的过程中，可能models经过多次改动最终migrations文件会到达0007_xxx.py。

所以从这方面看方式二还有点像manage.py的squashmigrations 命令，能够缩减migrations文件。这样一来缺点也相当明显，就是不太能保存对model所做的变更。举例来说，在两个版本之间你对model增加一个field，后来又由于某些原因删除了这个field，这种方式的migrations文件不会记录这些操作，而是什么改动也没有，就像没发生过一样。

可能还有一些没想到的方面，以后有机会补充。

----

2016.01.13更新

有一个问题是有时会写一些自定义的migration,而这个migration又是依赖某个特定版本的model的，如果你没有保存migrations的历史(也就是第二种方式)，就会很麻烦。因为在部署环境的migrations是部署和升级的时候生成的，你并不知道它最新的migrations版本是哪个，自定义的migration就很难确定依赖。因此，应该选择第一种方式。
