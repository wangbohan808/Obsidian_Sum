# 常用操作

## 一台电脑多个Github SSH

**可以同时创建多个，在访问github的时候，会进行逐个验证，直到遍历完毕，或者有一个符合要求的**

```bash
ssh-keygen -t ed25519 -C "your-personal-email@gmail.com"
# 保存文件时自定义名称，不要默认 id_ed25519
Enter file in which to save the key (~/.ssh/id_ed25519):
~/.ssh/id_github_personal
```
- `-t`后面接上的是不同的加密算法
- `-C`后续街上邮箱是为了方便从文件内容区分私钥文件


```bash
ssh-keygen -t ed25519 -C "work@company.com"
# 命名区分
Enter file in which to save the key (~/.ssh/id_ed25519):
~/.ssh/id_github_work
```

```bash
vim ~/.ssh/config
```

```bash
# 个人 GitHub 账号
Host github-personal
  HostName github.com
  User git
  IdentityFile ~/.ssh/id_github_personal
  IdentitiesOnly yes

# 公司 GitHub 账号
Host github-work
  HostName github.com
  User git
  IdentityFile ~/.ssh/id_github_work
  IdentitiesOnly yes
```
- `Host`：自定义别名，后续 Git 地址用这个别名代替 [github.com](https://link.wtturl.cn/?target=https%3A%2F%2Fgithub.com&scene=im&aid=582478&lang=zh "autolink")
- `HostName`：真实地址固定 [github.com](https://link.wtturl.cn/?target=https%3A%2F%2Fgithub.com&scene=im&aid=582478&lang=zh "autolink")
- `User git`：GitHub SSH 固定用户是 git，不可改
- `IdentityFile`：指定对应私钥路径
- `IdentitiesOnly yes`：强制只使用指定私钥，避免系统自动匹配其他密钥冲突

- 将公钥添加到对应服务器
```bash
# 个人公钥
cat ~/.ssh/id_github_personal.pub
# 公司公钥
cat ~/.ssh/id_github_work.pub
```

- 下载克隆代码有多种方式：
```bash
# 使用指定别名，密钥发送到服务器进行直接匹配
git clone git@github-work:company/pro.git

# 原生地址还是进行遍历
git clone git@github.com:xxx/repo.git
```

```bash
git clone https://github.com/wangbohan808/obsidian_celink.git

git clone git@github.com:wangbohan808/obsidian_celink.git
```


- 原本使用https克隆代码，修改为为SSH协议：
```bash
# 个人仓库
git remote set-url origin git@github-personal:personalName/demo.git

# 公司仓库
git remote set-url origin git@github-work:companyName/project.git
```


- 使用别名进行测试
```bash
# 测试个人密钥
ssh -T git@github-personal
# 测试工作密钥
ssh -T git@github-work
```


- 配置提交信息：
```bash
# 个人仓库内执行
git config user.name "wangbohan"
git config user.email "wangbohan808@163.com"

# 公司仓库内执行
git config user.name "公司姓名"
git config user.email "公司邮箱"

git config --global user.name "wangbohan"
git config --global user.email "wangbohan808@163.com"
```


## 拉取远程仓库另一个分支代码

```bash
# 1. 获取远程更新
git fetch origin

# 2. 操作后，发现所有远程分支拉取到本地远程仓库
git branch -r

# 3. 创建并切换到指定远程分支
git switch feature_m7a_iot
```


## 将本地代码推送到github仓库

```bash
# 配置用户名邮箱
git config --global user.name "wangbohan"
git config --global user.email "wangbohan808@163.com"
git config --list

# 网页端新建空仓库

# 初始化仓库，添加创建的空仓库
git init
git add .
git commit -m "初始化项目"
git remote add origin https://xxx.git

# 设置本地代码默认分支名，关联并推送代码
git branch -M main
git push -u origin main
```


## 企业级Git开发流程


