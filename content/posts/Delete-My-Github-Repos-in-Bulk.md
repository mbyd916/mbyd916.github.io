---
title: "批量删除自己的Github仓库"
date: 2019-11-25T18:22:18+08:00
draft: false

categories:
- 技巧
tags:
- Github删库
- 小工具  
keywords:
- Github批量删库
---

### 背景

fork了很多代码库，现在想批量删除，只保留自己创建的代码库。

解决方法：

使用Github提供的开发者API

### 1. 创建token

登录Github，按导航 “Settings/Developer settings”，切换到Tab “Personal access tokens”， 新生成一个token。选择 scope， “delelete_repo”。一次性操作，建议scope尽可能设置小范围，并且操作后删除该token。

### 2. 获取待删除repo列表

按导航“settings/repositories”，选取并复制所有repo，粘贴到一个文本文件，处理后得到一个自己准备删除的repo列表。

### 3. 请求删库API

```bash
# 创建的token
GITHUB_SECRET=""

# Curl请求API删除指定repo
function git_repo_delete(){
  curl -vL \
    -H "Authorization: token $GITHUB_SECRET" \
    -H "Content-Type: application/json" \
    -X DELETE https://api.github.com/repos/$1
}

# helper: 获取指定范围内的一个随机整数
function mimvp_randnum_file() {
    min=$1
    max=$2
    mid=$(($max-$min+1))
    num=$(head -n 20 /dev/urandom | cksum | cut -f1 -d ' ')
    randnum=$(($num%$mid+$min))
    echo $randnum
}

repos=$(cat <repo_file>)
for repo in $repos
do
  echo "deleting $repo ..."
  git_repo_delete "$repo"
  
  # 随机sleep，1～3秒
  rand=$(mimvp_randnum_file 1 3)
  sleep $rand
done
```

### **思考**

批量删库，Github为啥没有采取措施，如短时间拒绝访问，或者封号呢？

如果Github 账号被盗，风险还是很大的。。。
