---
type: 原子
status: 整理中
mastery: 3
tags:
  - 技术/工具
created: "2026-07-18"
updated: "2026-10-12"
---

# GitHub 使用

> [!info] 一句话结论
> 用 SSH 公私钥打通本机与 GitHub 的授权，再按「工作区 → 暂存区 → 本地仓库 → 远程仓库」四层模型完成日常提交、分支与协作。

## 要点
- 装好 Git 先配全局 `user.name` / `user.email`，仓库操作权限由 SSH 公私钥是否配对决定。
- 一台电脑多账号：用 `~/.ssh/config` 的 `Host` 别名 + `IdentitiesOnly yes` 区分。
- 日常主线：`status` → `add` → `commit` → `push`/`pull`，`log`/`diff`/`reset` 负责查看与回退。
- 分支用 `switch -c` 创建、`push -u` 关联远程；未追踪文件不拦截任何切换。
- 冲突先 `pull` 同步，处理 `<<<<<<<` 标记后 `add` + `commit` + `push`。
- 国内访问优先 SSH，必要时为 Git 单独配代理并把 `git://` 转成 `https://`。

## 安装与配置

### 安装与全局用户信息
官网下载 Git，默认下一步安装。随后配置提交者身份：

```bash
git config --global user.name "wangbohan"
git config --global user.email "wangbohan808@163.com"
```

查看当前配置（不同层级）：

```bash
git config --global --list
git config --local --list
cat .git/config
git config user.name
```

### 换行符
Linux 用 `LF`、Windows 用 `CRLF`，Git 有自动换行符转换机制，一般无需手动处理。

## SSH 密钥与认证

> [!info] 公私钥原理
> 1. 本机用邮箱生成一对公钥与私钥；
> 2. 公钥交给要操作的远程仓库保存配置；
> 3. 私钥留在本机，让本机优先使用；
> 4. 二者配对即可拉取、上传、修改该仓库。

### 生成密钥
推荐 ed25519 算法，`-C` 后接邮箱便于从内容区分私钥：

```bash
ssh-keygen -t ed25519 -f ~/.ssh/my_new_key -C "your_email@example.com"
```

设置权限（私钥必须 600）：

```bash
chmod 700 ~/.ssh
chmod 600 ~/.ssh/my_new_key
chmod 644 ~/.ssh/my_new_key.pub
chmod 644 ~/.ssh/config
```

查看本机已有密钥：`ls -al ~/.ssh`。

### 一台电脑多个 GitHub 账号
可同时创建多个密钥，访问 GitHub 时会逐个验证，直到匹配成功。

```bash
# 个人账号
ssh-keygen -t ed25519 -C "your-personal-email@gmail.com"
# 保存名称自定义，不要用默认 id_ed25519 → ~/.ssh/id_github_personal

# 公司账号
ssh-keygen -t ed25519 -C "work@company.com"
# → ~/.ssh/id_github_work
```

在 `~/.ssh/config` 中按别名区分：

```yaml
Host github-personal
  HostName github.com
  User git
  IdentityFile ~/.ssh/id_github_personal
  IdentitiesOnly yes

Host github-work
  HostName github.com
  User git
  IdentityFile ~/.ssh/id_github_work
  IdentitiesOnly yes
```

| 字段 | 含义 |
| --- | --- |
| `Host` | 自定义别名，clone 时用它替代 `github.com` |
| `HostName` | 真实地址，固定为 `github.com` |
| `User git` | GitHub SSH 固定用户名，不可改 |
| `IdentityFile` | 该别名使用的私钥路径 |
| `IdentitiesOnly yes` | 强制只用指定私钥，避免自动匹配冲突 |

把公钥内容添加到对应 GitHub 账号：

```bash
cat ~/.ssh/id_github_personal.pub
```

### 免重复输入密码（ssh-agent）
```bash
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/github_wangbohan808
ssh-add -l
```

持久化到 `~/.bashrc`，开终端自动加载：

```bash
if [ -z "$SSH_AUTH_SOCK" ]; then
  eval "$(ssh-agent -s)" > /dev/null
  ssh-add ~/.ssh/github_wangbohan808
fi
```

### 测试连接
```bash
ssh -T git@github-personal
ssh -T git@github-work
ssh -T git@github.com
```

## 日常指令

```bash
git init                          # 创建 .git，把普通目录变成仓库
git status                        # 查看工作区文件状态
git add .                         # 全部加入暂存区（或 git add <file>）
git add -u .                      # 只暂存已追踪文件的改动
git commit -m "描述这次提交"
git push -u origin main           # 首次推送并关联远程分支
git pull origin main              # 同步远程最新代码
git log --oneline                 # 单行查看历史提交
```

查看差异与回退：

| 命令 | 作用 |
| --- | --- |
| `git diff <file>` | 工作区 vs 暂存区/上次提交 |
| `git diff --staged <file>` | 暂存区 vs 最近一次提交 |
| `git diff HEAD <file>` | 工作区 vs 最近一次提交 |
| `git reset --hard <hash>` | 精准回退到某版本（会丢改动） |
| `git reset --hard HEAD` | 清空暂存区与工作区改动 |
| `git reset` / `git reset HEAD .` | 清空暂存区，保留工作区 |
| `git checkout .` | 丢弃工作区所有修改 |

## 分支管理

> [!note] 一个项目只需要一个工作区，但可以有多个分支；分支只是指向不同快照的指针。

```bash
git branch                                # 本地分支列表，* 为当前分支
git branch -r                             # 远程分支列表
git switch -c login-page                  # 基于当前分支创建并切换
git checkout -b feature/dust small-bag    # 基于指定分支创建
git push -u origin login-page             # 推送到远程并关联
```

重命名分支（先切到非目标分支）：

```bash
git branch -m old-name new-name
git push origin --delete old-name
git push origin new-name
```

删除本地与远程分支：

```bash
git fetch origin --prune
git branch -D feature/time-sharing
git push origin --delete feature/time-sharing
```

### 切换分支时的本地改动
- 两分支快照一致：可直接切换，本地改动保留。
- 快照不同且改动冲突：普通切换报错终止；`git checkout -f main` 强制切换会覆盖已追踪文件改动。
- 仅存在未追踪文件或被 `.gitignore` 忽略的文件：任何切换都不拦截。

## 冲突处理

> [!error] push 被拒（fetch first）
> 远程 `main` 存在本地没有的提交历史，Git 拒绝推送：
> ```
> ! [rejected]  main -> main (fetch first)
> hint: Updates were rejected because the remote contains work that you do not have locally.
> ```

处理原则：先让本地与最新远程同步 → 再把工作区改动落到本地仓库 → 最后推送。

```bash
git pull origin main --allow-unrelated-histories
```

冲突文件会被标记为：

```text
<<<<<<< HEAD
本机当前分支的内容
=======
远程 origin/main 分支的内容
>>>>>>> 46d0ddb
```

解决后加入暂存区并提交：

```bash
git add README.md
git commit -m "解决了 README 的冲突文件"
git push -u origin main
```

> [!tip] 冲突本质
> 未冲突部分进暂存区的一块区域，冲突文件进另一块；解决后 `git add` 把文件并回，全部解决即 `git commit`。

## 多人协作与 PR

- `git pull`：协作者把远程改动合并到本地。
- `pull request`：项目维护者审核后把「比较分支」的改动合并进「基础分支」。
- 分支首次建立时用 `push -u` 关联一次，后续 `push`/`pull` 默认在该关联分支上进行。

干净同步工作流的推荐顺序：

```bash
git stash                 # 暂存工作区改动，避免干扰
git pull origin main      # 同步远程，必要时解决冲突
git stash pop             # 恢复工作区改动，再解决一次冲突
```

## 仓库与远程管理

把本地代码推到 GitHub：

```bash
git init
git add .
git commit -m "初始化项目"
git remote add origin https://github.com/username/repository.git
git branch -M main
git push -u origin main
```

> [!warning] 网页端新建仓库后刷新一下再推送，避免推不上去。

远程仓库操作：

```bash
git remote -v                             # 查看已添加的远程仓库
git remote set-url origin git@github-work:company/project.git   # HTTPS 改为 SSH
```

- `origin`：远程仓库的默认名。
- `git@github.com:user/repo.git`：远程仓库 URL，用 SSH 协议；也可使用 HTTPS。

拉取远程的另一个分支：

```bash
git fetch origin
git branch -r
git switch feature_m7a_iot
```

## 常见问题

### 国内网络访问与代理
> [!note] SSH 比 HTTPS 稳定
> HTTPS 走 443，依赖复杂的 TLS 与代理环境，易被防火墙/杀软/代理/DNS 拦截；SSH 走 22 端口、链路更短、认证独立，常能一次成功。

Git 专属代理（只作用于 Git，不影响系统）：

```bash
git config --global http.https://github.com.proxy http://127.0.0.1:7897
git config --global https.https://github.com.proxy http://127.0.0.1:7897
```

部分仓库默认用被拦截的 `git://` 协议，统一转为 `https://`：

```bash
git config --global url."https://".insteadOf git://
```

环境变量代理与直连例外：

```bash
export HTTP_PROXY=http://127.0.0.1:7897
export HTTPS_PROXY=http://127.0.0.1:7897
export NO_PROXY=localhost,127.0.0.1,::1
```

> [!info] 优先级与接力
> 代理配置优先级：环境变量 `http_proxy` < Git 配置 `http.proxy` < 命令行参数，越精确越优先。
> 镜像与代理是「接力」而非冲突：镜像里有该文件就直连下载、不触发代理；镜像缺失才回退原始 URL，由代理代劳。

清理与验证代理残留：

```bash
git config --global --unset http.https://github.com.proxy
git config --global --unset https.https://github.com.proxy
git config --global --list | grep proxy
env | grep -i proxy
```

> [!warning] `unset` 只影响当前 shell。若代理写进了 `~/.bashrc`，新开终端仍会加载；要永久去除需编辑该文件。

> [!danger] `git config --global http.sslVerify false` 会关闭 Git 的 SSL 证书校验，仅可用于临时规避代理自建证书报错，长期使用存在中间人风险。

### 其他易错点
| 现象 | 处理 |
| --- | --- |
| 推送被拒 `(fetch first)` | 先 `git pull` 合并远程，再推送 |
| 切换分支报错 | 先提交或 `stash`，必要时 `checkout -f` 强制 |
| 需要停用某分支 | 本地 `branch -D`，远程 `push origin --delete` |
| 忘记当前密钥是否加载 | `ssh-add -l` 检查，未加载则重新 `ssh-add` |
