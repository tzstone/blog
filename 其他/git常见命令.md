# git 常见命令

## git revert

提交一个新版本, 将需要 revert 的版本内容反向修改回去. 版本会递增, 不影响之前提交的内容.

`git revert <commit>`: 撤销指定 commit 记录. 撤销会作为一次提交进行保存.

`git revert <commit1>..<commit2>`: 撤销 commit1 到 commit2 之间的所有 commit(前开后闭: 不包括 commit1, 包括 commit2).

举个例子，我们提交了 6 个版本，其中 3-4 包含了错误的代码需要被回滚掉。 同时希望不影响到后续的 5-6

```code
* 982d4f6 (HEAD -> master) version 6
* 54cc9dc version 5
* 551c408 version 4, harttle screwed it up again
* 7e345c9 version 3, harttle screwed it up
* f7742cd version 2
* 6c4db3f version 1
```

命令:

```code
git revert --no-commit f7742cd..551c408
git commit -a -m 'This reverts commit 7e345c9 and 551c408'
```

注意 revert 命令会对每个撤销的 commit 进行一次提交，`--no-commit` 后可以最后一起手动提交.

此时 Git 记录为:

```code
* 8fef80a (HEAD -> master) This reverts commit 7e345c9 and 551c408
* 982d4f6 version 6
* 54cc9dc version 5
* 551c408 version 4, harttle screwed it up again
* 7e345c9 version 3, harttle screwed it up
* f7742cd version 2
* 6c4db3f version 1
```

现在的 HEAD（8fef80a）就是我们想要的版本，把它 Push 到远程即可。

如果你像不确定是否符合预期，可以做一下 `diff` 来确认。 首先产生 version 4（551c408）与 version 6（982d4f6）的 diff，这些是我们想要保留的：

`git diff 551c408..982d4f6`

然后再产生 version 2（f7742cd）与当前状态（HEAD）的 diff：

`git diff f7742cd..HEAD`

如果 version 3, version 4 都被 version 6 撤销的话，上述两个 diff 记录是一致的。

### 合并冲突

#### 合并冲突后退出

`git revert --abort`: 当前的操作会回到指令执行之前的样子，相当于啥也没有干，回到原始的状态

#### 合并后退出，但是保留变化

`git revert --quit`: 该指令会保留指令执行后的车祸现场

#### 合并后解决冲突，继续操作

如果遇到冲突可以修改冲突，然后重新提交相关信息

```bash
git add .
git commit -m "提交的信息"
```

## git rebase

`git rebase -i [startpoint] [endpoint]`: 其中-i 的意思是--interactive，即弹出交互式的界面让用户编辑完成合并操作，[startpoint] [endpoint]则指定了一个编辑区间，如果不指定[endpoint]，则该区间的终点默认是当前分支 HEAD 所指向的 commit(注：该区间指定的是一个前开后闭的区间)

注意: 不要在 master 分支上 rebase, 否则会出现游离分支. 正常做法是比如在 dev 分支开发，然后需要的话在 dev 分支 rebase，然后再 merge 到 master.

`git rebase [startpoint] [endpoint] --onto [branchName`: 其中，[startpoint] [endpoint]仍然和上一个命令一样指定了一个编辑区间(前开后闭)，--onto 的意思是要将该指定的提交复制到哪个分支上。

## git reset

回滚到对应的 commit-id, 相当于删除了 commit-id 之后的所有提交,并且不会产生新的 commit-id 记录.

参见 [Git Reset 三种模式](https://www.jianshu.com/p/c2ec5f06cf1a)

## git diff

`git diff <commit>..<commit>`: 对比两个 commit 记录

参考资料:

[git](https://git-scm.com/docs)

[git 教程 --git revert 命令](https://www.cnblogs.com/ahzxy2018/p/14521922.html)

[git revert 的操作](https://www.jianshu.com/p/5e7ee87241e4)

[Git Reset 三种模式](https://www.jianshu.com/p/c2ec5f06cf1a)

[git rebase 详解](https://blog.csdn.net/weixin_42310154/article/details/119004977)

[【Git】rebase 用法小结](https://www.jianshu.com/p/4a8f4af4e803)
