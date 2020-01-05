---
layout: article
title: Git ignore的几种用法
tags: git
aside:
    toc: true
---

### 引言

Git是如今最主流的版本管理工具，经常应用在多人协作中，在git项目中，大部分文件都是上传到remote repository中并同步给所有人的。
不过有一些文件或目录并不想同步，举例如下：

* 一些中间过程产生的临时文件，比如.pyc文件，或者测试用例生成的report
* 与项目无关的文件，比如有一些IDE会自动产成一些目录或文件，这些目录或文件通常是隐藏的，且只适用于个人的IDE环境，并没有需要同步给其他人（每个人所使用的IDE不一定相同）
* 一些调试使用的配置文件

Git并不知道哪些文件属于上述范畴，对于新文件或者改动过的文件，git status统统会list出来，git add时也会将这些文件加入index中进而commit，如下面例子中的.pyc文件。

```bash
# git status
On branch master
Your branch is up-to-date with 'origin/master'.
Untracked files:
  (use "git add <file>..." to include in what will be committed)

	modules/__init__.pyc
	modules/libdemo.pyc

nothing added to commit but untracked files present (use "git add" to track)
```

那如何让git故意"看不见"这些文件，或是当他们不存在？Git提供了几种不同的方式，分别应用在不同的场合中。

### 1. 将文件加入.gitignore

如果这个文件或目录是不需要的临时文件，且项目中每个人都应该忽略它，我们就会把他加入项目根目录下的.gitignore文件中，比如：

```bash
*.pyc
```

加入后，git status就不会在untracked files中看到.pyc文件：

```bash
# cat .gitignore
*.pyc
# git status
On branch master
Your branch is up-to-date with 'origin/master'.
Untracked files:
  (use "git add <file>..." to include in what will be committed)

	.gitignore

nothing added to commit but untracked files present (use "git add" to track)
```

注意，按照惯例，.gitignore文件应该是需要大家公用的，也就是其中定义的ignore条目应该具有通用性。因此.gitiignore这个文件也需要push到remote branch，以便所有人同步配置。

```bash
# git add .gitignore
# git commit -m "Add .gitignore"
# git push origin master
# git status
On branch master
Your branch is up-to-date with 'origin/master'.
nothing to commit, working directory clean
```

Btw，Github针对主流的编程语言，提供了一个.gitignore的模板，移步[这里](https://github.com/github/gitignore)查看。

### 2. 将文件加入.git/info/exclude

第一种方法应用的场景是，添加进.gitignore中的文件或目录是每个人都需要忽略的；而某些场景下，有一些文件或目录仅在单个人的IDE环境中才出现，或是本地的某个临时文件并不想推送到远端，也就是说忽略这些文件并不是所有人的需求，仅在当前工作环境忽略，那么可以添加到.git/info/exclude这个文件中，而不用加到.gitignore。因为.git/本身是一个本地配置目录，并不会上传到远端。

下面例子是把.vscode这个IDE生成的目录加入到.git/info/exclude：

```bash
# git status
On branch master
Your branch is up-to-date with 'origin/master'.
Untracked files:
  (use "git add <file>..." to include in what will be committed)

	.vscode/

nothing added to commit but untracked files present (use "git add" to track)
# echo ".vscode" >> .git/info/exclude
# git status
On branch master
Your branch is up-to-date with 'origin/master'.
nothing to commit, working directory clean
```

无论是方法一还是方法二，都是对untracked file来设置，如果这个文件已经被提交过，那么这两种方法都无能为力了，此时无论是否加到.gitignore或.git/info/exclude中，都没有任何差异。补救的方法只能是将文件从git仓库中删除，再设置ignore。

### 3. 将文件标记为skip-worktree

试想这样一种场景，项目使用的配置文件用于生产环境或者公共环境，比如自动化测试项目中的配置文件包含测试环境的IP地址，但是开发者往往有自己的测试环境，与公共环境拥有不一样的IP地址，因此调试时往往会修改这个配置文件，改成自己的IP。这样，开发者在提交其他代码时，就需要很小心，以免把这个修改过的私人化的配置文件也一并提交上去。为了彻底避免这种意外，可以通过下面的方式，将这个配置文件设置为skip-worktree，这样git status就不会显示这个文件的变化，也就杜绝了被不小心加入提交行列的可能。

```bash
# 设置为skip-worktree
git update-index --skip-worktree config.json
# 取消设置
# git update-index --no-skip-worktree config.json
```

需要注意两点：

1. 这个配置是本地的，并不会提交给git仓库或是同步给其他人，因此需要每个开发者在各自的环境中执行。

2. 在配置后，如果本地的config.json不变，仅远端的config.json发生变更，则pull可以下拉并同步到最新版本。但如果本地和远端的config.json同时发生了变更，在pull或merge时，会报错：

   ```bash
   # git pull
   remote: Counting objects: 2, done.
   remote: Compressing objects: 100% (2/2), done.
   remote: Total 2 (delta 1), reused 0 (delta 0)
   Unpacking objects: 100% (2/2), done.
   From 10.1.12.10:chandler/git-ignore-demo
      26c1189..d8bc24f  master     -> origin/master
   Updating 26c1189..d8bc24f
   error: Your local changes to the following files would be overwritten by merge:
   	config.json
   Please, commit your changes or stash them before you can merge.
   Aborting
   ```

   此时，根据提示使用git stash，但发现git stash对标记为--skip-worktree的文件并不搭理：

   ```bash
   # git stash
   No local changes to save
   ```

   要处理这种情况，有两个办法：一种是先取消设置`git update-index --no-skip-worktree config.json`再处理冲突；另一种是先重命名该文件`mv config.json config.json.local`，等pull/merge之后再手动比对，更新本地配置到config.json。

如果要查看哪些文件被标记为skip-worktree，可以用`git ls-files -v`查看最前面的flag，S表示该文件被标记为skip-worktree：

```bash
# git ls-files -v .
H .gitignore
S config.json
H demo.py
H modules/__init__.py
H modules/libdemo.py
```

### 4. 将文件设置为assume-unchanged

Git还提供了一种方法来忽略文件变更，那就是将文件标记为assume-unchanged：

```bash
# 设置为skip-worktree
git update-index --assume-unchanged modules/*
# 取消设置
# git update-index --no-assume-unchanged modules/*
```

这种方法常常与上一种产生混淆，但两者应用场景正好相反。如果有一类文件，是不期待被本地更改的，比如一些第三方的SDK。可以将这类文件标记为assume-unchanged，这样git在去stat的时候会跳过它们，即使本地意外地更改这些文件（这种行为并不应该发生），也不会体现在git status里，所以也就不会被提交到git仓库。反之，第三种方法，是觉得该文件可能会被本地修改，但不希望修改被同步到远端。

同上一种方法，可以用`git ls-files -v`来查看哪些文件被标记为assume-unchanged，小写字母表示该文件被标记为assume-unchanged：

```bash
# git ls-files -v .
H .gitignore
S config.json
H demo.py
h modules/__init__.py
h modules/libdemo.py
```

Git设置这个flag的初衷是为了加速和优化git stat的性能，因此只有确定这些文件不被更改时，才选择这种方式。但是在git pull的时候，依然会去比对远端的差异，如果有更新，还是会同步本地文件，并且，__会偷偷将assume-unchanged的标记取消__（Git：说好的不会变呢，既然毁约就说明这个标记失效了，那么让我作废它吧。）

方法三和方法四的配置方式并不够直观，需要使用多次后方能加深理解。国外有位小哥早在八年前就做过一些[测试](https://fallengamer.livejournal.com/93321.html)，我摘录在这里供参考。

| **Operation**                                                | **File with assume-unchanged flag**                          | **File with skip-worktree flag**                             | **Comments**                                                 |
| ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| # File is changed both in local repository and upstream.<br /> **git pull** | Git wouldn’t overwrite local file. Instead it would output conflicts and advices how to resolve them. | Git wouldn’t overwrite local file. Instead it would output conflicts and advices how to resolve them. | Git preserves local changes anyway. Thus you wouldn’t accidently lose any data that you marked with any of the flags. |
| # File is changed both in local repository and upstream, trying to pull anyway.<br /> **git stash<br /> git pull** | Discards all local changes without any possibility to restore them. The effect is like ‘git reset --hard’. ‘git pull’ call will succeed. | Stash wouldn’t work on skip-worktree files. ‘git pull’ will fail with the same error as above. Developer is forced to manually reset skip-worktree flag to be able to stash and complete the failing pull. | Using skip-worktree results in some extra manual work but at least you wouldn’t lose any data if you had any local changes. |
| # No local changes, upstream file changed.<br /> **git pull** | Content is updated, flag is lost. ‘git ls-files -v’ would show that flag is modified to H (from h). | Content is updated, flag is preserved. ‘git ls-files -v' would show the same S flag as before the pull. | Both flags wouldn’t prevent you from getting upstream changes. Git detects that you broke assume-unchanged promise and choses to reflect the reality by resetting the flag. |
| # With local file changed.<br /> **git reset --hard**         | File content is reverted. Flag is reset to H (from h).       | File content is intact. Flag remains the same.               | Git doesn’t touch skip-worktree file and reflects reality (the file promised to be unchanged actually was changed) for assume-unchanged file. |

这边我也测试了一下（git version: 2.7.4），对其中一项存疑，第二行文件在本地和远端都发生变化时，如果文件被标记为assume-unchanged，在进行git stash的时候并不会"discard all local changes"，而是跟skip-worktree的文件一样不会有任何反应，如前文所述，开发者需要手动对这些文件做一些处理。不过在现实环境中不应该存在这种情况：被标记为assume-unchanged的文件发生了本地变更。

```bash
# git stash
No local changes to save
```

最后提一下，第四行测试，如果发生了本地变更，在`git reset --hard`之后，标记为assume-unchanged的文件会被打回原形(revert)被删除标记（Git：说好的不会变呢，怎么又在本地变了呢，幸好被我reset发现了，给你取消标记了）；而标记为skip-worktree的文件不受影响，保持本地变更。

### 总结

| METHOD                   | DESCRIPTION                                                  | BEST USE CASES                                               | SCOPE  |
| ------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ | ------ |
| .gitignore               | Prevents files from being added to the repository.           | Local settings, compiler output, test results, etc.          | Global |
| .git/info/exclude        | Prevents local files from being added to the repository.     | Local settings, compiler output, test results, etc.          | Local  |
| skip-worktree setting    | Prevents local changes from being committed to an existing file. | Shared files that will have local overwrites, like config files. | Local  |
| assume-unchanged setting | Allows Git to skip files that won’t be changed for performance optimization. | Files and folders that a developer won’t touch.              | Local  |

### 参考

[1]: https://automationpanda.com/2018/09/19/ignoring-files-with-git/	"IGNORING FILES WITH GIT"
[2]: https://fallengamer.livejournal.com/93321.html	"FallenGameR's blog"
[3]: https://stackoverflow.com/questions/13630849/git-difference-between-assume-unchanged-and-skip-worktree
