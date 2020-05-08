---
layout: article
title: Pytest的测试顺序
tags: python
key: test_order_of_pytest
---

## 引言

自动化环境使用Allure来展示pytest的测试报告，对其进行日志分析时，常常发现一个困扰：测试的顺序似乎总是让人捉摸不透，有时候像是测试标题字符排序，有时候又是根据测试函数定义的先后顺序排序；而且pytest对子目录的执行顺序也跟Allure这边（无论是按order还是按name排序）的展示不一致。结果是，如果有case因为环境不ready而失败时，无法快速定位到该失败case上一条执行的case。

![](/asserts/posts/test_order_of_pytest/test_order_of_pytest_1.png)

## 问题

针对这个问题，文档中找到的说法是：

> Pytest runs the test in the same order as they are found in the test module.

这里只提到了同一个python文件(module)中，测试时是按test定义的顺序，但未提到目录顺序，感觉起来是按字母排序，但是又跟Allure这边按name排序看到的不一致。所以有了第一个问题：

- **问题一：pytest对目录的执行顺序是怎样的？**

针对同一个module中的执行顺序，找了几个module，在Allure UI点击按照order排序，比对python文件，发现大部分都如此；但也有个别文件的执行顺序并不是按照test定义的顺序，而是按字母顺序。所以有了第二个问题：

- **问题二：单个module中的test执行顺序为何有的是按定义顺序，有的是按字母顺序？**

这里顺便提一下，如果想查看pytest执行测试的顺序，跑一遍花费时间较久，可以在运行pytest时带上`--collect-only`参数，这样仅会收集并打印这些测试用例，而不会去运行具体的用例。

## 原因

网上搜来终觉浅，绝知此事要溯源。为了弄清楚细节，还是要看源码，所幸python代码并不难找。

~~~python
  # https://github.com/pytest-dev/pytest/blob/.../src/_pytest/python.py 
  def collect(self):
        if not getattr(self.obj, "__test__", True):
            return []

        # NB. we avoid random getattrs and peek in the __dict__ instead
        # (XXX originally introduced from a PyPy need, still true?)
        dicts = [getattr(self.obj, "__dict__", {})]
        for basecls in self.obj.__class__.__mro__:
            dicts.append(basecls.__dict__)
        seen = set()
        values = []
        for dic in dicts:
            # Note: seems like the dict can change during iteration -
            # be careful not to remove the list() without consideration.
            for name, obj in list(dic.items()):
                if name in seen:
                    continue
                seen.add(name)
                res = self._makeitem(name, obj)
                if res is None:
                    continue
                if not isinstance(res, list):
                    res = [res]
                values.extend(res)

        def sort_key(item):
            fspath, lineno, _ = item.reportinfo()
            return (str(fspath), lineno)

        values.sort(key=sort_key)
        return values
      
    def reportinfo(self) -> Tuple[Union[py.path.local, str], int, str]:
        # XXX caching?
        obj = self.obj
        compat_co_firstlineno = getattr(obj, "compat_co_firstlineno", None)
        if isinstance(compat_co_firstlineno, int):
            # nose compatibility
            file_path = sys.modules[obj.__module__].__file__
            if file_path.endswith(".pyc"):
                file_path = file_path[:-1]
            fspath = file_path  # type: Union[py.path.local, str]
            lineno = compat_co_firstlineno
        else:
            fspath, lineno = getfslineno(obj)
        modpath = self.getmodpath()
        assert isinstance(lineno, int)
        return fspath, lineno, modpath
~~~

这里可以看到，pytest收集测试用例的时候，会优先根据文件的路径进行排序，其次是行号。那回到第一个问题，如果文件是按照路径名字排序，那为何跟在Allure上选择按name排序的结果不同呢？原因是Allure按name排序时并不区分大小写，如第一张图所示，对于testcase.Function_Test.case_10_S3下的文件，排序方式如下：

test_S3_a_new_S3_pool > test_S3_bucket_access_logging > test_S3_Default_S3_pool > test_S3_quota
{: .success}

而python对于字符串的排序其实是区分大小写的，大写字母会优先于小写字母，也就是这样：

test_S3_Default_S3_pool > test_S3_a_new_S3_pool > test_S3_bucket_access_logging > test_S3_quota
{: .success}

对照日志的时期戳，这也正是pytest执行测试的顺序。如果想让Allure和pytest的顺序一致，可以全部改用小写字母，或是为每个文件夹名字添加数字编号，安排测试顺序。

到此就解决了第一个问题。

再看第二个问题，也就是单个module中test的执行顺序，从上面的代码看到，对于相同路径的文件来说，排序的方式是根据test所在的行号，那为何还有一些特殊用例是按照名字排序执行的呢？

继续在源码中寻找原因，原来在我们的测试用例中，有一部分由于历史原因，使用了unittest的编写方式，也就是testcase继承自unittest。对于这些testcase，pytest会使用如下方式来收集用例：

~~~python
# https://github.com/pytest-dev/pytest/blob/.../src/_pytest/unittest.py
    def collect(self):
        from unittest import TestLoader

        cls = self.obj
        if not getattr(cls, "__test__", True):
            return

        skipped = getattr(cls, "__unittest_skip__", False)
        if not skipped:
            self._inject_setup_teardown_fixtures(cls)
            self._inject_setup_class_fixture()

        self.session._fixturemanager.parsefactories(self, unittest=True)
        loader = TestLoader()
        foundsomething = False
        for name in loader.getTestCaseNames(self.obj):
            x = getattr(self.obj, name)
            if not getattr(x, "__test__", True):
                continue
            funcobj = getimfunc(x)
            yield TestCaseFunction.from_parent(self, name=name, callobj=funcobj)
            foundsomething = True

        if not foundsomething:
            runtest = getattr(self.obj, "runTest", None)
            if runtest is not None:
                ut = sys.modules.get("twisted.trial.unittest", None)
                if ut is None or runtest != ut.TestCase.runTest:
                    # TODO: callobj consistency
                    yield TestCaseFunction.from_parent(self, name="runTest")

~~~

可以看到，这里是调用unittest中的TestLoader()来收集测试用例，那么顺序也就取决于unittest加载testcase的顺序了，事实上unittest默认会按照字母顺序来加载，这种方式会带来一些困扰，不仅是因为跟pytest的排序方式不一致，而且，当我们打算让测试用例按照我们期望的顺序去执行时，需要小心地给用例起名，这显然不能作为一条编写准则。

这里多说一句，网上有不少人觉得既然是unittest，那么每条testcase之间本就不应该存在依赖关系，所以不用关心测试用例的先后顺序；但是在某些场合，特别是在做系统测试或是集成测试时，有一些前置步骤花费时间较久，我们可能会巧妙安排测试顺序来减少重复步骤，节约测试的总时间。举个简单例子，如果我们有两条测试用例，一条是开启某个功能，另一条是关闭这个功能，如果先跑关闭功能的测试用例，那么势必要在测试前先开启功能，这其实与开启功能的测试用例重复了。因此为了缩短测试时间，我们可以先测试开启功能的用例，在测试完成后，再执行下一条关闭功能的测试用例。

因为在我们环境中算是历史遗留问题，而且只有少量unittest继承过来的用例，因此解决办法是将这部分改写成pytest建议的格式，舍弃unittest。如果一定要继续使用unittest，可以尝试修改TestLoader的默认排序函数：

~~~b
sortTestMethodsUsing
    Function to be used to compare method names when sorting them in getTestCaseNames() and all the loadTestsFrom*() methods. The default value is the built-in cmp() function; the attribute can also be set to None to disable the sort.
~~~

## 总结

1. pytest执行测试用例的顺序：先将用例对应的python文件路径按字符串排序，分别执行不同的testsuite，在同一文件/module下的用例按照在文件中定义的顺序来执行。
2. unittest执行测试用例的顺序：同一module中的测试用例按照字符串进行排序执行。
