# 环境部署

## MCP部署常用Node语言

|场景|优先语言|理由|
|---|---|---|
|模型训练、微调、大模型推理|Python|PyTorch 生态，科学计算库完善|
|RAG 向量检索、Embedding 算法|Python|生态成熟|
|MCP Server、本地 Agent、工具调度|Node|stdio 管道、子进程稳定，跨平台，npx|
|网页 AI 应用前端|JS/TS|浏览器原生|
|简单 MCP 小工具、轻量脚本|Python|代码简单，上手快|


| 概念       | Node 生态             | Python 生态           | 类比关系              |
| -------- | ------------------- | ------------------- | ----------------- |
| 运行时程序    | `node`              | `python`            | 近似对等              |
| 包管理器     | `npm`               | `pip`               | 近似对等              |
| 一次性临时执行包 | `npx`               | `pipx`              | 近似对等（Python 不是内置） |
| 环境隔离     | fnm/nvm（切换 Node 版本） | pyenv（切换 Python 版本） | 对等：切换解释器本体        |
| 项目依赖隔离   | node_modules        | venv / conda        | 思路相似，但底层实现不一样     |


## 安装部署流程

1. 安装解释器与包管理器
```powershell
node -v
npm -v
```
- 如果上述没有安装，就进行下面的安装

```powershell
winget install OpenJS.NodeJS.LTS

# 可能还要取消禁止，给一下权限
Set-ExecutionPolicy -Scope CurrentUser RemoteSigned
```


- 直接使用`brew`包管理器
```mac
brew install node
```


- 使用`curl`下载`node`的解释器管理工具
```mac
curl -fsSL https://fnm.vercel.app/install | bash

fnm install 24
fnm default 24
```


2. 直接使用`agent`下载部署“逆向工程的MCP”
```text
Install REA and connect it to this coding agent using npx rea-agents@latest setup. Show me the setup plan for approval, then verify the installation.
```



**理解MCP协议以及使用：**

- MCP Client（Claude Code）= 老板（有大脑，做决策）
- MCP Server（rea-agents）= 外包工程师（单独的人，会逆向分析）
- MCP 协议 = 两个人之间固定的沟通话术规范
- 配置文件 = 通讯录，记录怎么联系外包工程师（启动命令）
- stdin/stdout 管道 = 两人之间的专线电话

流程：

1. 安装 setup：老板把外包的联系方式存入通讯录
2. 老板上班（启动 Client），看通讯录，拨通电话（拉起子进程）
3. 外包自我介绍：我能做哪些活（上报工具列表）
4. 老板遇到任务，打电话发指令，外包在本地干活，把结果回话
5. 老板下班，挂断电话，外包下班，人消失。