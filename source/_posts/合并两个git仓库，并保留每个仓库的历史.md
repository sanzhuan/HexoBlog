title: "合并两个git仓库，并保留每个仓库的历史"
date: 2015-05-20 20:08:07
tags: git
---


## 问题：有两个git仓库 repository1和repository2,想把repository2合并到repository1，并在repository1中保留repository2的所有分支历史

<!-- more --> 

## 方案：  
1.在repository1中添加远程分支,，跟踪repository2，并把repository2全部分支拉到repository1中，命令为：

``` bash
$ git  remote add other /path/to/repository2
$ git fetch other 
```

2.第一步执行完后，在repository1工作目录中.git/refs/remotes/ 文件夹下有个other文件夹，ohter文件夹里是所有的repository2分支，因此只要把这些分支设置成repository1分支即可，命令为:

``` bash
   for branch in `ls .git/refs/remotes/ohter`
   do
     if test "${branch}"="master"
       then
       git checkout -b othermaster other/${branch}
     else
       git checkout  -b ${branch} other/${branch}
     fi
  done
```

### 参考：
http://stackoverflow.com/questions/1683531/how-to-import-existing-git-repository-into-another