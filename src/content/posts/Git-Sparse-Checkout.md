---
title: Git稀疏检出
description: 如果我们工作的git仓库过大，克隆整个仓库也许没有什么必要，我们可以只克隆一部分。
image: https://blogimgs-1309485105.cos.ap-nanjing.myqcloud.com/Cover/Linux/3.jpg
category: Tools
published: 2023-04-02
tags: [Git]
---
------

当我们使用`git clone`命令的时候，我们是默认完整克隆整个仓库的。但是，如果我们要工作的仓库过于庞大，而我们本身又只需要修改其中的一部分文件的话，克隆整个仓库就会显得非常不必要，此时我们可以使用git的稀疏检出功能，来只克隆和修改部分文件。

## 做法

假设我们的仓库名称为project，地址在`https://github.com/<YourName>/project.git`，我们可以这样做：

1. 创建一个空目录来存放项目：

    ```shell
    $ mkdir project
    $ cd project
    ```
2. 初始化git仓库，添加远端仓库地址

    ```shell
    $ git init
    $ git remote add origin https://github.com/<YourName>/project.git
    ```
3. 配置本地仓库为稀疏检出模式：

    ```shell
    $ git config core.sparsecheckout true
    ```
4. 在`.git/info`下，创建一个`sparse-checkout`文件，然后写入需要的文件和目录名，假设我们只需要修改`README.md`，目录`dir1`下的所有文件，以及目录`dir2`下的`main.cc`，那么在`sparse-checkout`文件下，可以这样写：

    ```plaintext
    README.md
    dir1/*
    dir2/main.cc
    ```
5. 接着，拉取远端代码即可

    ```shell
    $ git pull origin master
    ```

## 更新稀疏检出路径

如果后续需要修改`.git/info/sparse-checkout`来检出额外的文件或目录，可以直接编辑该文件并添加新行：

```shell
$ echo "dir3/*" >> .git/info/sparse-checkout
```

完成编辑后，运行以下命令刷新稀疏检出的内容：

```shell
$ git read-tree -mu HEAD
```

## 注意一下gitignore

因此，不要将`.gitignore`文件内容用于稀疏检出，也就是不要把`.gitignore`写入到`sparse-checkout`文件里面。因为gitignore和sparse-checkout的作用正好相反。如果你讲`.gitignore`写到`sparse-checkout`里面，你会发现`sparse-checkout`将不会起作用。
