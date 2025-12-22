---
title: Hexo新建个人博客
date: 2025-03-15 20:20:37
tags: Hexo
---

## 环境搭建

**环境需求**

1. Node.js
2. Git
3. Hexo

### 安装Node.js

### 安装Git

### 安装Hexo

## Github连接设置

根目录下：Git Bash Here

### 设置邮箱和用户名

```shell
git config --global user.name "GitHub 用户名"
git config --global user.email "GitHub 邮箱"
```

### 创建SSH密钥

输入 `ssh-keygen -t rsa -C "GitHub 邮箱"`，然后一路回车（使用默认配置）

进入 [C:\Users\用户名\.ssh] 目录（要勾选显示“隐藏的项目”），用记事本打开公钥 id_rsa.pub 文件并复制里面的内容。

登陆 GitHub ，进入 Settings 页面，选择左边栏的 SSH and GPG keys，点击 New SSH key。

Title 随便取个名字，粘贴复制的 id_rsa.pub 内容到 Key 中，点击 Add SSH key 完成添加。

不想随意添加全局的SSH密钥的话可以使用仓库的deploy keys。

验证连接：打开 Git Bash，输入 `ssh -T git@github.com` 出现 “Are you sure……”，输入 yes 回车确认。

### 创建Github Pages仓库

GitHub 主页右上角加号 -> New repository：

- Repository name 中输入 `用户名.github.io`
- 勾选 “Initialize this repository with a README”
- Description 选填

填好后点击 Create repository 创建。

## 本地配置

### 博客初始化

```shell
hexo init      # 初始化
npm install    # 安装组件
hexo g   # 生成页面
hexo s   # 启动预览
```

### 博客部署

首先**安装 hexo-deployer-git**：

```text
npm install hexo-deployer-git --save
```

然后**修改 _config.yml** 文件末尾的 Deployment 部分，修改成如下：

```text
deploy:
  type: git
  repository: git@github.com:用户名/用户名.github.io.git
  branch: main
```

**注意：现在的github默认分支为main**

完成后运行 `hexo d` 将网站上传部署到 GitHub Pages。

完成！这时访问我们的 GitHub 域名 `https://用户名.github.io` 就可以看到 Hexo 网站了。

### 博客更新

## 主题设置

## 远程协作

使用hexo d进行github的推送的时候仅仅会将pages相关的文件推送，hexo的环境相关文件不会推送，如果本地环境更换或者丢失，或者对博客进行协作的更新的话，会非常麻烦。

### 同步原理

将hexo生成的静态文件仍然保留再main分支上，在我们执行hexo d的时候，仍然是更新main分支。

将hexo的源文件 放在source分支上。

在更新的时候，先更新main分支，再更新source分支。需要协同的时候，git clone 从source分支下载源文件，安装好环境之后就可以重新生成静态页面。

### 源主机操作

创建source分支，设置为默认分支。

打包原始文件

1. clone该仓库到本地（默认分支ssource）
2. 删除文件夹内除.git之外的其他文件
3. 在hexo原文件下，复制除了.deploy_git之外的所有其他文件，到当前文件中

**注意：**1.现在clone下来的文件夹内应该有个`.gitignore文件`，用来忽略一些不需要的文件，表示这些类型文件不需要git。如果没有，右键新建，内容如下：

```text
.DS_Store
Thumbs.db
db.json
*.log
node_modules/
public/
.deploy*/
```

2.如果原文件有主题，将主题的.git文件也要一并删除，因为git不可以嵌套上传。

### 重新推送

```shell
git add .
git commit –m add_branch
git push
```

### 更新操作

**更新静态页面**

```shell
hexo clean
hexo g
hexo d
```

**更新资源**

```
git add .
git commit –m add_branch
git push
```

### 常见问题

#### 无法push推送

ERROR: The key you are authenticating with has been marked as read only.
fatal: Could not read from remote repository.

Please make sure you have the correct access rights
and the repository exists.

由于deploy key权限不正确，需要设定为**读写权限**

#### 推送到错误分支

未将source设定为默认分支；

解决方法：1.将source设定为默认分支 2.新建本地的source分支，并将其关联到远程source分支

```
git checkout -b source
git add .
git commit -m "source branch"
git push --set-upstream origin source
```

## 绑定域名

## 参考链接

[使用 Hexo+GitHub 搭建个人免费博客教程（小白向） - 知乎](https://zhuanlan.zhihu.com/p/60578464)

[Hexo在多台电脑上提交和更新_hexo两个设备-CSDN博客](https://blog.csdn.net/K1052176873/article/details/122879462)
