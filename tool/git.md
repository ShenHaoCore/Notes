Git 指南 - 从入门到精通
===============

* * *

## 1. 代码提交与同步命令

### 1.1 流程图

#### 第一步：工作区与仓库保持一致

#### 第二步：文件增删改，变为已修改状态

#### 第三步：`git add`，变为已暂存状态

```bash
$ git status 
$ git add --all              # 当前项目下的所有更改
$ git add .                  # 当前目录下的所有更改
$ git add xx/xx.py xx/xx2.py # 添加某几个文件
```

#### 第四步：`git commit`，变为已提交状态

```bash
$ git commit -m "<这里写commit的描述>"
```

#### 第五步：`git push`，变为已推送状态

```bash
$ git push -u origin master # 第一次需要关联上
$ git push                  # 之后再推送就不用指明应该推送的远程分支了
```

#### 第六步：查看分支

```bash
$ git branch     # 查看本地仓库的分支
$ git branch -a  # 查看本地仓库和远程仓库的所有分支 
```

## 2. 代码撤销与同步命令

### 2.1 已修改但未暂存

```bash
$ git diff                         # 列出所有的修改
$ git diff xx/xx.py xx/xx2.py      # 列出某几个文件的修改
$ git checkout                     # 撤销项目下所有的修改
$ git checkout .                   # 撤销当前文件夹下所有的修改
$ git checkout xx/xx.py xx/xx2.py  # 撤销某几个文件的修改
$ git clean -f                     # 撤销新增的文件（untracked 状态）
$ git clean -df                    # 撤销新增的文件和文件夹（untracked 状态）
```

### 2.2 已暂存但未提交

```bash
$ git diff --cached    # 显示暂存区和本地仓库的差异
$ git reset            # 暂存区的修改恢复到工作区
$ git reset --soft     # 回到已修改状态，修改的内容仍然在工作区中
$ git reset --hard     # 回到未修改状态，清空暂存区和工作区
```

### 2.3 已提交但未推送

```bash
$ git diff <branch-name1> <branch-name2>    # 比较两个分支之间的差异
$ git diff master origin/master             # 查看本地仓库与远程仓库的差异
$ git reset --hard origin/master            # 回退到与远程仓库一致的状态
$ git reset --hard HEAD^                    # 回退到上一个版本
$ git reset --hard <hash code>              # 回退到任意版本
```

### 2.4 已推送到远程

```bash
$ git push -f origin master   # 强制覆盖远程分支
$ git push -f                 # 如果之前已经用 -u 关联过，则可省略分支名 
```

### 3. 其他常用命令

### 3.1 初始化与关联远程仓库

```bash
$ git init                          # 初始化仓库 
$ git remote add origin <url>       # 关联远程仓库 
$ git remote add coding <url>       # 关联多个远程仓库 
$ git remote -v                     # 查看关联的仓库地址 
$ git clone <url>                   # 克隆远程仓库到本地 
$ git remote set-url origin <url>   # 修改远程仓库地址
```

### 3.2 配置信息

```bash
$ git config --list                                                             # 查看当前配置 
$ git config --global user.name "<name>"                                        # 配置用户名 
$ git config --global user.email "<email>"                                      # 配置邮箱 
$ git config --global alias.logg "log --graph --decorate --abbrev-commit --all" # 设置别名
```

### 3.3 分支操作

```bash
$ git checkout -b <new-branch-name> # 新建并切换分支
$ git branch                        # 查看本地分支
$ git branch -a                     # 查看本地和远程分支
$ git checkout <branch-name>        # 切换到现有分支
$ git pull origin <branch-name>     # 更新代码
$ git stash                         # 暂存工作区修改
$ git stash pop                     # 恢复暂存的修改
```

### 3.4 版本回退与前进

```bash
$ git status                                       # 查看当前状态 
$ git log                                          # 查看历史版本 
$ git log --graph --decorate --abbrev-commit --all # 更好看的日志 
$ git reflog                                       # 查看最近的操作记录 
$ git reset --hard <hash>                          # 回退到指定版本 
$ git reset --hard HEAD^                           # 回退到上一个版本 
$ git reset --hard HEAD^^                          # 回退到上上个版本 
$ git commit --amend                               # 修改上一次提交的信息 
```

## 4. 分支命名与环境对应

| 分支        | 功能              | 环境  | 可访问 |
| --------- | --------------- | --- | --- |
| `master`  | 主分支，稳定版本        | PRO | 是   |
| `develop` | 开发分支，最新版本       | DEV | 是   |
| `feature` | 开发分支，实现新特性      |     | 否   |
| `test`    | 测试分支，功能测试       | FAT | 是   |
| `release` | 预上线分支，发布新版本     | UAT | 是   |
| `hotfix`  | 紧急修复分支，修复线上 bug |     | 否   |

* * *

## 5. Git Commit Message 规范

### 5.1 标准格式

```Plaintext
<type>(<scope>): <subject>

<BLANK LINE>

<body>

<BLANK LINE>

<footer>
```

#### 字段说明

* **type**: 提交类型（如 `feat`, `fix`, `docs`, `style`, `refactor`, `perf`, `test`, `chore` 等）
* **scope**: 可选，本次 commit 波及的范围
* **subject**: 简明扼要地阐述本次 commit 的主旨
* **body**: 详细描述本次变更的动机和内容
* **footer**: 描述与之关联的 issue 或 break change

### 5.2 示例

```Plaintext
feat(user_module): 添加用户登录功能

新增了用户登录的接口调用和前端表单验证逻辑。
解决了之前未实现用户身份验证的问题。

Closes #123
```
