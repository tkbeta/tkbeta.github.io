---
layout: article
title: Linux中的内部命令和外部命令
tags: shell
key: internal_external_commands
---

今天使用python的subprocess.Popen去执行系统命令时，碰到一个奇怪的问题，精简后如下所示：

```bash
$ echo -e

$ python -c "import subprocess; subprocess.Popen('echo -e', shell=True)"
-e
$ python -c "import subprocess; subprocess.Popen('echo -n', shell=True)"
```

即-e参数不生效，被当作一个普通文本输出到屏幕，但-n参数可以生效，os.system也有同样的问题：

```bash
$ python -c "import os; os.system('echo -e')"
-e
```

换了Ubuntu和CentOS均是一样的问题，为什么`echo`放到python里面去执行就是不一样的行为？

后来想到也许是shell的类型不同，果然将终端切换到sh，就发现在sh下`echo -e`不支持`-e`语法：

```bash
sh:~$ echo -e
-e
sh:~$ which echo
/bin/echo
```

`which echo`得到的程序路径和`bash`下查看的一样，那为什么`echo`指令却行为不一？

一番谷歌后有了大致的结论，原来Linux中的命令分内部命令和外部命令，内部命令(internal command或称builtin command)是直接built在shell中的，运行速度更快，因为执行命令时不需要查找PATH以及产生子进程。

不过，有些命令同时拥有内部和外部两种实现，即操作系统既提供了built-in的实现，又提供了外部的utility(通常在/bin/或/usr/bin下)，比如`echo`, `kill`等，为什么同样的命令会有不同的实现并存，找到如下可能的原因：

1. Under a variety of circumstances, you run an executable directly and not through a shell, for instance, find doesn't use a shell, it runs /bin/echo.

   ```
   find src -name 'abc*' -type f -exec echo mv -nv {} dest/ \;
   ```

2. some shells don't have an `echo` builtin. This is not actually required.

3. because the [POSIX specification](http://pubs.opengroup.org/onlinepubs/9699919799/utilities/contents.html) says there has to be one.

值得注意的是，`which`只是用来查找可执行文件的存放路径，并不能用来查询命令的实际来源，一般通过`type`来或`type -a`查看命令实际使用的来源和全部的来源。

```bash
$ type echo
echo is a shell builtin
$ type -a echo
echo is a shell builtin
echo is /bin/echo
$ type kill
kill is a shell builtin
$ type -a kill
kill is a shell builtin
kill is /bin/kill
kill is /usr/bin/kill
```

可以看到echo和kill都有internal和external两种命令，其中internal(builtin)的命令优先级更高。

```sh
# bash
$ type echo
echo is a shell builtin
# sh
$ type echo
echo is a shell builtin
```

在sh和bash下分别执行`type echo`看到都是shell builtin，所以其实两者都不是用的`/bin/echo`，而是各自builtin的命令。区别在于，bash的builtin echo支持-e语法，而sh的 builtin echo不支持。

回到最初的问题，如果要让python使用`echo -e`的语义，可以用`bash -c`的方式调用：

```python
import subprocess

def bash_command(cmd):
    subprocess.Popen(['/bin/bash', '-c', 'echo -e'])
```

最后，如果想知道Linux下哪些是内部指令，可以用`help`命令，找个bash环境试试看吧。

### 参考

1. [Why is echo a shell built in command?](https://unix.stackexchange.com/questions/1355/why-is-echo-a-shell-built-in-command)
2. [Why is there a /bin/echo and why would I want to use it?](https://askubuntu.com/questions/960822/why-is-there-a-bin-echo-and-why-would-i-want-to-use-it)
3. [Internal and External Commands in Linux](https://www.geeksforgeeks.org/internal-and-external-commands-in-linux/)
